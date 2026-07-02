# BloodHound & PlumHound

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
```

---

## Collecting Data

### bloodhound-python (From Kali - Recommended)

```sh
# Full collection (noisiest but most data)
bloodhound-python -d corp.local -u jdoe -p Password1 -ns <DC-IP> -c all

# DC-only collection (quieter - avoids noisy session enumeration)
bloodhound-python -d corp.local -u jdoe -p Password1 -ns <DC-IP> -c DCOnly

# With hash instead of password
bloodhound-python -d corp.local -u jdoe --hashes :<NTLM-HASH> -ns <DC-IP> -c all
```

| Flag | Notes |
|---|---|
| `-d` | Domain FQDN |
| `-u` | Any valid domain user |
| `-p` | Plaintext password |
| `--hashes` | Format `:NTLM` (colon prefix) |
| `-ns` | DC IP - used for DNS resolution |
| `-c` | Collection method |

**Collection methods:** `all` (high noise), `DCOnly` (low - recommended for real engagements), `Session` (very noisy), `ACL` (medium)

### Via NetExec
```sh
nxc ldap <DC-IP> -u jdoe -p Password1 --bloodhound --collection All --dns-server <DC-IP>
```

### SharpHound (From Windows)
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

| Query | Why |
|---|---|
| **Find Shortest Paths to Domain Admins** | Your attack path |
| **Find All Domain Admins** | Target for impersonation/creds |
| **Find Principals with DCSync Rights** | Can dump all hashes without NTDS.dit |
| **Find Computers Where Domain Users are Local Admin** | Lateral movement targets |
| **Shortest Paths from Kerberoastable Users** | Where does a cracked TGS lead? |
| **Find AS-REP Roastable Users** | No preauth - easy hashes |
| **Shortest Paths to Unconstrained Delegation** | Machines that get TGTs forwarded |

---

## Reading the Graph

**Nodes:** Blue = User, Green = Computer, Yellow = Group, Red outline = High-value target

**Edges:**

| Edge | Meaning | Attack |
|---|---|---|
| `MemberOf` | User is in this group | Check group privileges |
| `AdminTo` | Local admin on this computer | PTH/lateral movement |
| `GenericAll` | Full control | Reset password, add to group |
| `GenericWrite` | Modify object attributes | Set SPN for Kerberoasting |
| `WriteDACL` | Modify permissions | Give yourself GenericAll |
| `WriteOwner` | Take ownership | Take ownership -> modify DACL |
| `ForceChangePassword` | Reset password | Reset -> authenticate as them |
| `DCSync` | Replicate domain (dump all hashes) | `lsadump::dcsync` |
| `HasSession` | User has session on machine | Token impersonation target |

**Right-click any edge -> Help -> shows the exact attack for that relationship.**

---

## Finding Your Path

1. Search for your current user in the search bar
2. Right-click -> "Mark User as Owned"
3. Run "Shortest Paths to Domain Admins from Owned Principals"
4. Follow the chain of edges

---

## Plumhound (Turn BloodHound Data Into a Task List)

```sh
sudo python3 PlumHound.py -x tasks/default.tasks -p <neo4j-password>
# Output: reports/ folder with HTML showing prioritized attack paths
```

---

## OPSEC Notes

- `Session` collection queries every domain computer - very noisy
- `DCOnly` only hits the DC with LDAP queries - looks like normal AD traffic
- bloodhound-python from Kali is quieter than SharpHound on target
- Do not run BloodHound collection repeatedly - once is enough
