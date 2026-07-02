# Web Enumeration

---

## Gobuster

### Directory Brute Force
```sh
gobuster dir -u http://<ip> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50

# With file extensions
gobuster dir -u http://<ip> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x .php,.html,.js,.txt

# Output to file
gobuster dir -u http://<ip> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -o gobuster.txt
```

### Subdomain Brute Force
```sh
gobuster dns -d domain.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### VHost Brute Force
```sh
gobuster vhost -u http://<ip> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

---

## Nikto (Web Vulnerability Scanner)
```sh
nikto -h http://<ip>
nikto -h https://<ip> -ssl
nikto -h http://<ip> -p 8080
```

---

## Dirbuster
GUI tool - good for targeted searches with custom file extensions.

Wordlists location: `/usr/share/wordlists/dirbuster/`
> Adding file extensions increases scan time significantly.

---

## Feroxbuster (Faster Alternative to Gobuster)
```sh
feroxbuster -u http://<ip> -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

---

## Subdomain Enumeration

```sh
# assetfinder (passive)
assetfinder --subs-only domain.com | tee subdomains.txt

# Resolve live subdomains
cat subdomains.txt | httprobe | tee alive.txt
```

---

## Burp Suite

### Setup
```sh
# Start via terminal
burpsuite
```

### Quick Workflow
1. Configure browser proxy -> 127.0.0.1:8080
2. Intercept -> Forward through interesting requests
3. Send to Repeater for manual testing
4. Use Intruder for fuzzing / brute force

---

## Useful Wordlists (SecLists)

| Purpose | Path |
|---|---|
| Common directories | `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` |
| Common files | `/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt` |
| Subdomains | `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` |
| Passwords | `/usr/share/wordlists/rockyou.txt` |
| Usernames | `/usr/share/seclists/Usernames/top-usernames-shortlist.txt` |

---

## What to Look For

- Login pages -> default creds, SQLi, auth bypass
- Upload functionality -> webshell upload
- File includes -> LFI/RFI
- URL parameters -> SQLi, path traversal
- Version numbers in headers/footers -> searchsploit for CVEs
- `/robots.txt` and `/sitemap.xml` -> hidden paths
- Source code comments -> credentials, internal paths
