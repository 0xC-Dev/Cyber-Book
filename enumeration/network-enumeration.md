# Network Enumeration

## Standard Nmap Workflow

### Step 1 - Fast Port Discovery
```sh
nmap -T4 -p- --min-rate 5000 <ip> -oN ports.txt
```

### Step 2 - Deep Scan on Open Ports
```sh
# Substitute open ports found above
nmap -sC -sV -p 22,80,443 <ip> -oN detailed.txt
```

### Step 3 - UDP (if needed)
```sh
nmap -sU --top-ports 20 <ip>
```

---

## enum4linux (Quick AD/SMB Info)
```sh
# Gets users, shares, groups, OS info
enum4linux -a <ip>
```

---

## Port / Service Quick Reference

| Port | Service | Quick Enum |
|---|---|---|
| 21 | FTP | `ftp <ip>`, try anonymous login |
| 22 | SSH | Version fingerprint, check for default creds |
| 25/465/587 | SMTP | `nmap -sC -p25`, `nc <ip> 25` EHLO |
| 53 | DNS | Zone transfer, `dig axfr @<ip> domain.com` |
| 80/443 | HTTP/HTTPS | See [Web Enumeration](web-enumeration.md) |
| 88 | Kerberos | AD present - try kerbrute |
| 111 | RPC | `rpcinfo -p <ip>` |
| 135/139/445 | SMB/NetBIOS | See [Service Enumeration](service-enumeration.md) |
| 161/162 | SNMP | `snmpwalk`, community string check |
| 389/636 | LDAP | `ldapsearch`, `ldapdomaindump` |
| 2049 | NFS | `showmount -e <ip>`, mount and browse |
| 3389 | RDP | Check for BlueKeep (CVE-2019-0708) |
| 5985/5986 | WinRM | Evil-WinRM if you have creds |
| 8080/8443 | Alt HTTP | Web app, admin panels |

---

## DNS Enumeration

```sh
# Zone transfer (can dump all DNS records)
dig axfr @<dns-server> domain.com

# Reverse lookup
dig -x <ip> @<dns-server>

# Host lookup
host domain.com <dns-server>

# Brute force subdomains
gobuster dns -d domain.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

---

## SNMP Enumeration

```sh
# Walk SNMP tree (community string often "public")
snmpwalk -c public -v1 <ip>
snmpwalk -c public -v2c <ip>

# Enumerate with onesixtyone (community string brute force)
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <ip>
```

---

## NFS Enumeration

```sh
# List NFS shares
showmount -e <ip>

# Mount a share
mkdir /tmp/nfs
mount -t nfs <ip>:/share /tmp/nfs

# Check for no_root_squash (can write as root)
cat /etc/exports
```

---

## RPC Enumeration

```sh
rpcinfo -p <ip>

# If 111 open, check for NFS
rpcclient -U "" <ip>
  > enumdomusers
  > enumdomgroups
  > querydominfo
```
