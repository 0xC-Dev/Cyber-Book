# BloodHound - Full Reference

BloodHound maps AD relationships as a graph. You collect raw data from the domain, import it, then query attack paths visually. It answers "how do I get from this user to Domain Admin?"

**Requires Neo4j running first** - BloodHound is just the frontend.

---

## Setup (Every Session)

```sh
# Step 1 - Start Neo4j database
sudo neo4j console
# Access at http://localhost:7474 - set password on first run

# Step 2 - Start BloodHound GUI
sudo bloodhound
# Login with your neo4j credentials
```

---

## Collecting Data

### bloodhound-python (From Kali - Recommended)

Run from your Kali machine through the network. Nothing touches disk on the target.

```sh
# Full collection (noisiest but most data)
bloodhound-python -d corp.local -u jdoe -p Password1 -ns <DC-IP> -c all

# DC-only collection (quieter - avoids noisy session enumeration)
bloodhound-python -d corp.local -u jdoe -p Password1 -ns <DC-IP> -c DCOnly

# With hash instead of password
bloodhound-python -d corp.local -u jdoe --hashes :<NTLM-HASH> -ns <DC-IP> -c all
```

**Every flag:**

| Flag | What it takes | Notes |
|---|---|---|
| `-d` | Domain FQDN | `corp.local` |
| `-u` | Username | Any valid domain user |
| `-p` | Password | Plaintext |
| `--hashes` | NTLM hash | Format `:NTLM` (colon prefix) |
| `-ns` | Name server IP | DC IP - used for DNS resolution |
| `-c` | Collection method | See table below |
| `-dc` | Specific DC to query | Optional - `dc01.corp.local` |

**Collection methods (`-c`):**

| Method | What it collects | Noise |
|---|---|---|
| `all` | Everything below | High |
| `DCOnly` | Users, groups, trusts, ACLs, GPOs from DC only | Low |
| `Session` | Who's logged into what machine right now | High - queries every computer |
| `ACL` | Permission relationships (GenericAll, WriteDACL etc.) | Medium |
| `Trusts` | Domain trust relationships | Low |
| `Default` | Session + ACL + LocalAdmin + Trusts | Medium |

### Via NetExec (One-liner)
```sh
nxc ldap <DC-IP> -u jdoe -p Password1 --bloodhound --collection All --dns-server <DC-IP>
```

### SharpHound (From Windows - if on domain machine)
```powershell
.\SharpHound.exe -c all --outputdirectory C:\temp
# Generates a zip file - transfer to Kali and import
```

---

## Importing Data

1. Open BloodHound GUI
2. Click **Upload Data** button (top right)
3. Drag and drop the generated `.json` files (or zip from SharpHound)
4. Wait for import - progress bar at bottom

---

## Key Queries (Run These Every Time)

In the search bar -> click the query icon:

| Query | Why |
|---|---|
| **Find Shortest Paths to Domain Admins** | Most important - your attack path |
| **Find All Domain Admins** | See who has DA - target for impersonation/creds |
| **Find Principals with DCSync Rights** | Can dump all hashes without touching NTDS.dit |
| **Find Computers Where Domain Users are Local Admin** | Lateral movement targets |
| **Shortest Paths from Kerberoastable Users** | If you crack a TGS, where does it lead? |
| **Find AS-REP Roastable Users** | No preauth required - easy hashes |
| **Shortest Paths to Unconstrained Delegation** | Machines that get TGTs forwarded to them |

---

## Reading the Graph

**Nodes:**
- Blue circle = User
- Green circle = Computer  
- Yellow = Group
- Red outline = High-value target

**Edges (the lines between nodes) - what they mean:**

| Edge | Meaning | Attack |
|---|---|---|
| `MemberOf` | User is in this group | Check group privileges |
| `AdminTo` | User/group has local admin on this computer | PTH/lateral movement |
| `GenericAll` | Full control over the object | Reset password, add to group |
| `GenericWrite` | Can modify object attributes | Set SPN for Kerberoasting |
| `WriteDACL` | Can modify permissions on object | Give yourself GenericAll |
| `WriteOwner` | Can take ownership | Take ownership -> modify DACL |
| `ForceChangePassword` | Can reset the password | Reset -> authenticate as them |
| `AllExtendedRights` | Can do anything extended | Includes force change password |
| `DCSync` | Can replicate domain (dump all hashes) | `lsadump::dcsync` |
| `HasSession` | A user has a session on this machine | Token impersonation target |

**Right-click any edge -> Help -> shows the exact attack for that relationship.**

---

## Finding Your Path

1. Search for your current user in the search bar
2. Right-click -> "Mark User as Owned"
3. Run "Shortest Paths to Domain Admins from Owned Principals"
4. Follow the chain of edges - each edge tells you the exact attack to run

---

## Plumhound (Turn BloodHound Data Into a Task List)

```sh
# Must have BloodHound running with data loaded and neo4j up
sudo python3 PlumHound.py -x tasks/default.tasks -p <neo4j-password>

# Output: reports/ folder with HTML showing prioritized attack paths
```

---

## OPSEC Notes

- `Session` collection queries every domain computer - very noisy, generates lots of 4624/4634 events
- `DCOnly` only hits the DC with LDAP queries - looks like normal AD traffic
- bloodhound-python from Kali through the network is quieter than running SharpHound on a target
- Don't run BloodHound collection repeatedly - once is usually enough
