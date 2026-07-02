# Passive Recon & OSINT

> No direct contact with target systems - all open-source discovery.

---

## Email & People Discovery

| Tool | Usage | Notes |
|---|---|---|
| **hunter.io** | Find company emails | 50-100 free searches/month |
| **phonebook.cz** | Emails, domains, URLs | Broad search |
| **clearbit** | Chrome extension, email enrichment | 100 free searches |
| **HaveIBeenPwned** | Check breached credentials | Check found emails |
| **Breach-Parse** | Parse breach databases | Offline tool |

---

## Domain & Subdomain Discovery

```sh
# Passive subdomain enum
assetfinder --subs-only target.com

# Certificate transparency logs (no active requests)
curl -s "https://crt.sh/?q=%.target.com&output=json" | jq '.[].name_value' | sort -u

# Sublist3r
sublist3r -d target.com -o subdomains.txt
```

---

## Web Fingerprinting (Passive)

| Tool | Usage |
|---|---|
| **Wappalyzer** | Browser extension - tech stack detection |
| **WhatWeb** | CLI fingerprinting tool |
| **BuiltWith** | Web-based stack analysis |
| **Netcat** | Banner grabbing |

```sh
# Banner grabbing
nc -nv <ip> 80
```

---

## Location & Physical OSINT

- Satellite images (Google Maps, Bing Maps)
- Job postings -> reveals tech stack, tools used, department sizes
- LinkedIn -> employee names, titles, org structure, tech mentions
- GitHub -> leaked credentials, internal tooling, config files
- Shodan -> internet-exposed services on target IPs

---

## Google Dorking

```
site:target.com filetype:pdf
site:target.com intitle:"index of"
site:target.com ext:config OR ext:env OR ext:xml
"@target.com" filetype:xls
```

---

## Target Validation

Before attacking, validate the target:
- Verify IP ownership (WHOIS, ARIN/RIPE lookup)
- Confirm domains resolve to target IPs
- Ensure everything is within the agreed scope

```sh
whois target.com
nslookup target.com
host target.com
```
