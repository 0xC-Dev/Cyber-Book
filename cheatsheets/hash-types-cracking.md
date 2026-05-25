# Hash Types & Cracking Reference

---

## Hashcat Mode Reference

| Mode | Hash Type | Example Start |
|---|---|---|
| `0` | MD5 | `5f4dcc3b5aa765d61d8327deb882cf99` |
| `100` | SHA1 | `5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8` |
| `1000` | NTLM | `31d6cfe0d16ae931b73c59d7e0c089c0` |
| `1800` | sha512crypt ($6$) | `$6$rounds=5000$...` |
| `3200` | bcrypt ($2*$) | `$2y$10$...` |
| `5600` | NTLMv2 (Responder) | `user::domain:...` |
| `13100` | Kerberoast TGS-REP | `$krb5tgs$23$...` |
| `18200` | AS-REP (Kerberos) | `$krb5asrep$23$...` |
| `21000` | DCC2 (Domain Cache) | `$DCC2$10240#user#...` |

---

## Quick Crack Commands

```sh
# NTLM (SAM/NTDS)
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt

# NTLMv2 (Responder output)
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt

# Kerberoast
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt

# AS-REP Roast
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# DCC2
hashcat -m 21000 dcc2.txt /usr/share/wordlists/rockyou.txt --force

# MD5
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# Show already cracked
hashcat -m <mode> hashes.txt /usr/share/wordlists/rockyou.txt --show

# Rule-based attack (more coverage)
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

---

## John the Ripper

```sh
# Auto-detect and crack
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt

# NTLM
john hashes.txt --format=NT --wordlist=/usr/share/wordlists/rockyou.txt

# Show cracked
john --show hashes.txt

# Unshadow (combine passwd + shadow)
unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## Hash Identification

```sh
# hashid — identify unknown hashes
hashid '<hash>'
hashid -m '<hash>'    # Shows hashcat mode number

# hash-identifier (interactive)
hash-identifier
```

---

## Password Spraying

```sh
# Hydra — SSH
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://<ip>

# Hydra — HTTP form
hydra -l admin -P rockyou.txt <ip> http-post-form "/login:username=^USER^&password=^PASS^:Invalid"

# Hydra — SMB
hydra -L users.txt -P passwords.txt smb://<ip>

# NetExec spray (AD)
nxc smb <cidr> -u users.txt -p 'Password123' --continue-on-success
# Avoid lockouts: check policy with ldapdomaindump first
```

---

## Extract NT Hash from Secretsdump Output

Secretsdump format: `user:RID:LM:NT:::`
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

Extract only NT hashes for hashcat:
```sh
cat secretsdump_output.txt | cut -d: -f4 > nt_hashes.txt
```

---

## Common Default Credentials

| Service | Username | Password |
|---|---|---|
| Router admin | admin | admin, password |
| MySQL | root | (blank), root |
| MSSQL | sa | (blank), sa |
| Tomcat | tomcat | tomcat, s3cret |
| Jenkins | admin | admin |
| WordPress | admin | admin |
| Plex | admin | admin |

```sh
# Wordlist of default creds
/usr/share/seclists/Passwords/Default-Credentials/
```
