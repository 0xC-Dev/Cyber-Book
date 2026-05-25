# Service Enumeration

---

## SMB (Port 139 / 445)

### SMB Version Reference
| Version | Windows | Notes |
|---|---|---|
| CIFS | NT 4.0 | NetBIOS interface |
| SMB 1.0 | Windows 2000 | Direct TCP 445, vulnerable to EternalBlue |
| SMB 2.0 | Vista / 2008 | Improved signing |
| SMB 3.1.1 | Windows 10 / 2016 | AES-128 encryption |

### Nmap SMB Scripts
```sh
# Check SMB signing (unsigned = vulnerable to relay)
nmap --script=smb2-security-mode.nse -p445 <ip/cidr>

# Full SMB enumeration
nmap -sC -sV -p139,445 <ip>
```

### NetExec SMB Enumeration
See [NetExec](../tools/netexec.md) for full nxc reference.

```sh
# Host discovery and SMB signing check
nxc smb <ip/cidr>

# Null session share enum
nxc smb <ip> -u '' -p '' --shares

# Guest session
nxc smb <ip> -u 'guest' -p '' --shares

# RID brute force (user enum)
nxc smb <ip> -u '' -p '' --rid-brute

# Password spray
nxc smb <ip> -u users.txt -p 'Password123' --continue-on-success
```

### smbclient
```sh
# Null session list shares
smbclient -L //<ip> -N

# Connect to share (null)
smbclient -U 'a' -N //<ip>/<share>

# Download all files recursively
smbclient //<ip>/<share> -N -c 'recurse ON; prompt OFF; mget *'
```

### enum4linux-ng
```sh
enum4linux-ng -A <ip>
```

---

## FTP (Port 21)

### Anonymous Login
```sh
ftp <ip>
# Username: anonymous
# Password: (blank or anonymous@)
```

### Download Everything
```sh
wget -m --no-passive ftp://anonymous:anonymous@<ip>
```

### Useful Nmap FTP Scripts
```sh
nmap -sC -sV -p21 <ip>
# Key scripts: ftp-anon, ftp-vsftpd-backdoor, ftp-bounce
```

### vsFTPd Config (if accessible)
```sh
cat /etc/vsftpd.conf | grep -v "#"
# Watch for: anonymous_enable=YES, write_enable=YES
```

---

## SMTP (Port 25 / 465 / 587)

```sh
# Banner grab
nc -nv <ip> 25

# EHLO to list supported commands
EHLO example.com

# Nmap scripts
nmap -sC -p25,465,587 <ip>

# User enum via VRFY
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t <ip>
```

---

## LDAP (Port 389 / 636)

```sh
# Anonymous query (if allowed)
ldapsearch -H ldap://<ip> -x -b "dc=domain,dc=com"

# Authenticated dump
ldapsearch -H ldap://<ip> -x -D "domain\user" -W -b "dc=domain,dc=com"

# Full domain dump
python3 -m ldapdomaindump ldaps://<ip> -u 'DOMAIN\user' -p 'Password'
```

---

## NFS (Port 2049)

```sh
# List exports
showmount -e <ip>

# Mount
mkdir /mnt/nfs && mount -t nfs <ip>:/share /mnt/nfs

# no_root_squash = you can write files as root, escalate
```

---

## WinRM (Port 5985 / 5986)

```sh
# Connect with Evil-WinRM (need valid creds or hash)
evil-winrm -i <ip> -u <user> -p <password>
evil-winrm -i <ip> -u <user> -H <NTLM-hash>
```

---

## RDP (Port 3389)

```sh
# Connect
xfreerdp /u:<user> /p:<pass> /v:<ip>
xfreerdp /u:<user> /pth:<hash> /v:<ip>

# Check for BlueKeep
nmap -sV --script rdp-vuln-ms12-020 -p3389 <ip>
```
