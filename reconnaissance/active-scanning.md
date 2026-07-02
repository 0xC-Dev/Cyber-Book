# Active Scanning

> Direct network contact - ensure you are authorized before running these.

---

## Host Discovery

```sh
# ARP scan entire subnet (layer 2 - very reliable on local network)
netdiscover -r 192.168.1.0/24

# Nmap ping scan (no port scan)
nmap -sn 192.168.1.0/24

# ICMP echo only
nmap -PE 192.168.1.0/24

# TCP SYN ping (port 80 default)
nmap -PS80 192.168.1.0/24

# Scan for hosts with any open port (fast)
sudo nmap -v --min-rate 10000 192.168.20.0-254 | grep open
```

---

## Nmap Port Scanning

### Quick Initial Scan
```sh
# Fast scan all ports, then detailed on open ones
nmap -T4 -p- --min-rate 5000 <ip> -oN initial.txt
```

### Full Detailed Scan
```sh
# All ports, version detection, default scripts, OS fingerprint
nmap -T4 -p- -A <ip> -oN full.txt
```

### Targeted Service Scan
```sh
# Once you know open ports, run scripts on them
nmap -sC -sV -p 22,80,443,445 <ip>
```

### Nmap Flag Reference

| Flag | Purpose |
|---|---|
| `-sS` | TCP SYN stealth scan (default, needs root) |
| `-sT` | TCP connect scan (no root needed, louder) |
| `-sU` | UDP scan (very slow - use carefully) |
| `-sV` | Version detection |
| `-sC` | Run default scripts |
| `-A` | Version + scripts + OS + traceroute |
| `-p-` | All 65535 TCP ports |
| `-p 80,443` | Specific ports |
| `-T4` | Speed (0=paranoid -> 5=insane, T4 is good balance) |
| `-Pn` | Skip host discovery (treat as up) |
| `-O` | OS detection |
| `--min-rate 5000` | Minimum packet rate |
| `-oN file.txt` | Normal output to file |
| `-oA basename` | All formats (normal, XML, grepable) |

### UDP Scan (Common Ports)
```sh
# UDP is slow - target common ports only
nmap -sU -p 53,67,68,69,123,161,162,500 <ip>
```

---

## Rustscan (Faster Initial Scan)

```sh
# Fast port discovery, pass to nmap for details
rustscan -a <ip> -- -A

# With specific nmap flags
rustscan -a <ip> -- -sC -sV -oN rustscan.txt
```

---

## SSH Older Algorithms

```sh
# Connect to SSH servers with deprecated key exchange algorithms
ssh <ip> -oKexAlgorithms=+diffie-hellman-group1-sha1

# Older host key type
ssh <ip> -oHostKeyAlgorithms=+ssh-rsa
```
