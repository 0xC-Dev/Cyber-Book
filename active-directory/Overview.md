# Active Directory Overview

---

## Physical Components

### Domain Controller (DC)
- Most critical piece of AD
- Hosts the AD DS directory store
- Provides authentication and authorization
- Replicates to other DCs in the domain/forest
- Admins manage user accounts and resources through it

### AD DS Data Store (Ntds.dit)
- Database storing: users, groups, security descriptors, **password hashes**
- Default location: `%SystemRoot%\NTDS\ntds.dit`
- Only accessible through DC processes — dumping it = owning the domain
- See [Dumping NTDS.dit](domain-compromise/dumping-ntds.md)

---

## Logical Components

### AD DS Schema
- Blueprint/rules for all objects in the directory
- Defines object types and their attributes
- Enforces creation rules

### Domains
- Group and manage objects in an organization
- Admin boundary for policy application
- Replication boundary between DCs
- Authentication and authorization boundary

### Trees
- Hierarchy of domains (like subdomains on a network)
- Child domains inherit trust from parent

### Forest
- Collection of domain trees
- **Enterprise Admins** and **Schema Admins** are forest-wide
- Compromise one domain → potential forest-wide access

### Organizational Units (OUs)
- Containers for objects (users, computers, groups)
- Group Policy Objects (GPOs) applied at OU level

### Trusts
- Allow users to access resources in another domain
- Types: One-way, Two-way, Transitive, Non-transitive

---

## Authentication Protocols

| Protocol | Notes |
|---|---|
| **NTLM** | Challenge-response, hash-based. Vulnerable to relay and pass-the-hash |
| **Kerberos** | Ticket-based. Default in AD. Vulnerable to Kerberoasting, Golden Ticket |
| **LDAP** | Directory queries. Use for enumeration |

### Attack Flow
```
Initial access
    ↓
LLMNR/SMB Relay → Get NTLMv2 hash → Crack or relay
    ↓
Valid credentials
    ↓
Enumerate with BloodHound / PowerView
    ↓
Attack paths:
- Kerberoasting → crack TGS hash
- Pass the Hash → lateral movement
- Token impersonation → DA if lucky
    ↓
Domain Admin → Dump NTDS.dit → All hashes → Golden Ticket
```

---

## Key AD Attack Reference

| Attack | Notes |
|---|---|
| LLMNR Poisoning | [initial-access/llmnr-poisoning.md](initial-access/llmnr-poisoning.md) |
| SMB Relay | [initial-access/smb-relay.md](initial-access/smb-relay.md) |
| IPv6 (mitm6) | [initial-access/ipv6-mitm6.md](initial-access/ipv6-mitm6.md) |
| BloodHound | [enumeration/bloodhound.md](enumeration/bloodhound.md) |
| PowerView | [enumeration/powerview.md](enumeration/powerview.md) |
| Pass the Hash | [post-compromise/pass-the-hash.md](post-compromise/pass-the-hash.md) |
| Kerberoasting | [post-compromise/kerberoasting.md](post-compromise/kerberoasting.md) |
| Token Impersonation | [post-compromise/token-impersonation.md](post-compromise/token-impersonation.md) |
| Mimikatz | [post-compromise/mimikatz.md](post-compromise/mimikatz.md) |
| NTDS.dit Dump | [domain-compromise/dumping-ntds.md](domain-compromise/dumping-ntds.md) |
| Golden Ticket | [domain-compromise/golden-ticket.md](domain-compromise/golden-ticket.md) |
| Skeleton Key | [domain-compromise/skeleton-key.md](domain-compromise/skeleton-key.md) |
| DSRM | [domain-compromise/dsrm.md](domain-compromise/dsrm.md) |
| Custom SSP | [domain-compromise/custom-ssp.md](domain-compromise/custom-ssp.md) |
