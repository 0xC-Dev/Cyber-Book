# Summary

* [Introduction](README.md)

## Methodology
* [Pentest Process](methodology/pentest-process.md)

## Reconnaissance
* [Passive Recon & OSINT](reconnaissance/passive-recon-osint.md)
* [Active Scanning](reconnaissance/active-scanning.md)

## Enumeration
* [Network Enumeration](enumeration/network-enumeration.md)
* [Service Enumeration](enumeration/service-enumeration.md)
* [Web Enumeration](enumeration/web-enumeration.md)

## Active Directory
* [Overview](active-directory/overview.md)
* **Initial Access**
  * [Kerbrute, Spray & AS-REP](active-directory/initial-access/kerbrute-spray-asrep.md)
  * [LLMNR Poisoning](active-directory/initial-access/llmnr-poisoning.md)
  * [SMB Relay](active-directory/initial-access/smb-relay.md)
  * [IPv6 / mitm6](active-directory/initial-access/ipv6-mitm6.md)
  * [Pre2k & AD Misconfigurations](active-directory/initial-access/pre2k-misconfigurations.md)
  * [Gaining Shell Access](active-directory/initial-access/gaining-shell-access.md)
* **Enumeration**
  * [BloodHound & PlumHound](active-directory/enumeration/bloodhound.md)
  * [PowerView](active-directory/enumeration/powerview.md)
  * [LDAPDomainDump](active-directory/enumeration/ldapdomaindump.md)
* **Post-Compromise**
  * [Pass the Hash](active-directory/post-compromise/pass-the-hash.md)
  * [Kerberoasting](active-directory/post-compromise/kerberoasting.md)
  * [Token Impersonation](active-directory/post-compromise/token-impersonation.md)
  * [Mimikatz](active-directory/post-compromise/mimikatz.md)
  * [ADCS Attacks](active-directory/post-compromise/adcs-attacks.md)
  * [ACL Abuse](active-directory/post-compromise/acl-abuse.md)
  * [Delegation Attacks](active-directory/post-compromise/delegation-attacks.md)
  * [LNK File Attack](active-directory/post-compromise/lnk-file-attack.md)
  * [GPP cPassword](active-directory/post-compromise/gpp-cpassword.md)
  * [GPO Abuse](active-directory/post-compromise/gpo-abuse.md)
* **Domain Compromise**
  * [Dumping NTDS.dit](active-directory/domain-compromise/dumping-ntds.md)
  * [Golden Ticket](active-directory/domain-compromise/golden-ticket.md)
  * [Silver Ticket](active-directory/domain-compromise/silver-ticket.md)
  * [Diamond Ticket](active-directory/domain-compromise/diamond-ticket.md)

## Post-Exploitation
* [Linux Privilege Escalation](post-exploitation/linux-privesc.md)
* [Windows Privilege Escalation](post-exploitation/windows-privesc.md)

## Web Attacks
* [SQL Injection](web-attacks/sql-injection.md)
* [Path Traversal & LFI](web-attacks/path-traversal.md)
* [Command Injection](web-attacks/command-injection.md)
* [File Upload Bypass](web-attacks/file-upload-bypass.md)
* [Authentication Bypass](web-attacks/authentication-bypass.md)
* [SSRF](web-attacks/ssrf.md)
* [XXE](web-attacks/xxe.md)
* [SSTI](web-attacks/ssti.md)

## Tools
* [NetExec (nxc)](tools/netexec.md)
* [Impacket Suite](tools/impacket.md)
* [Mimikatz](tools/mimikatz.md)
* [Rubeus](tools/rubeus.md)
* [BloodHound](tools/bloodhound.md)
* [Responder](tools/responder.md)
* [PrintSpoofer & Potato Attacks](tools/printspoofer-potatoes.md)
* [MSSQL Attacks](tools/mssql.md)
* [Tunneling (Ligolo-ng & Chisel)](tools/tunneling.md)
* [Metasploit](tools/metasploit.md)

## Exploits & CVEs
* [Buffer Overflow (x86 Windows)](exploits/buffer-overflow.md)
* [Finding & Using Exploits](exploits/finding-exploits.md)
* [EternalBlue (MS17-010)](exploits/eternalblue.md)
* [PrintNightmare (CVE-2021-1675)](exploits/printnightmare.md)

## Cheatsheets
* [Master Command Reference](cheatsheets/master-command-reference.md)
* [Reverse Shells](cheatsheets/reverse-shells.md)
* [Hash Types & Cracking](cheatsheets/hash-types-cracking.md)
* [File Transfers](cheatsheets/file-transfers.md)
