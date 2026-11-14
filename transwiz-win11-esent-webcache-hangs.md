# SOLVED: Transwiz Windows 10 → Windows 11 migration — Chrome downloads stuck at 100%, Outlook and Settings hang (ESENT / WinINet WebCache)

**Confirmed field fix, 14 November 2026.** This documents one Windows 10 → Windows 11 Active Directory user-profile migration using Transwiz. It is a troubleshooting lead, **not** evidence that every Transwiz migration causes the issue.

## Symptoms

On the migrated **domain user account only** (a different account on the same Windows 11 machine worked normally):

- Chrome and Edge downloads reached **100%** but did not finish in the browser.
- The files were already present and correctly written to disk.
- Outlook hung at **“Loading Profile”** or account setup; other Office components could be slow.
- Windows Settings (`ms-settings:`) took minutes to open.
- Internet Properties / **LAN settings** and some printer-related dialogs were extremely slow.
- Some installers or applications appeared to hang.

Ordinary filesystem permissions and the computer's AD domain trust checked out. The problem was broader than Chrome or Outlook.

## Quick diagnosis: check the Application event log

Look for **ESENT** events associated with **`WebCacheLocal`**, especially:

| Event ID | Error | Seen in this case |
| --- | --- | --- |
| **490** | **-1032** / Windows error **5** | Could not open `WebCacheV01.dat` for read/write: **Access denied** |
| **494** | **-1216** | Database recovery expected a database at a path that was not available |
| **454** | **-1907** | Unexpected database recovery error |

In this case, the errors were recurring approximately every ten seconds.

To review recent relevant errors (run in PowerShell; change the time range as needed):

```powershell
Get-WinEvent -FilterHashtable @{
    LogName      = 'Application'
    ProviderName = 'ESENT'
    StartTime    = (Get-Date).AddHours(-4)
} -MaxEvents 200 -ErrorAction SilentlyContinue |
Where-Object { $_.Id -in 454, 490, 494 -and $_.Message -like '*WebCacheLocal*' } |
Select-Object TimeCreated, Id, Message |
Format-List
```

**Important clue:** This migration had an unsuccessful first import using one profile path, then a successful import using a different suffixed profile path, e.g.:

```text
C:\Users\example.user          # first, abandoned import directory
C:\Users\example.user.DOMAIN   # active migrated profile
```

ESENT's error messages referenced the **old** path, not the active profile path. This was a strong indicator of inconsistent WebCache recovery state, but the precise mechanism by which the path was retained was not independently proven.

## Fix that worked: back up and regenerate the user's WinINet WebCache

**Only do this if your evidence points to the WinINet WebCache.** The cache is user-specific. Do **not** delete the entire user profile, Credential Manager, certificate stores, browser profile or unrelated application data.

1. **Fully sign out** the affected user. Locking or disconnecting the session is insufficient.
2. Sign in with a **different administrator account**.
3. Confirm the affected user's profile is not loaded:

   ```powershell
   Get-CimInstance Win32_UserProfile |
       Where-Object LocalPath -eq 'C:\Users\example.user.DOMAIN' |
       Select-Object LocalPath, Loaded
   ```

   Proceed only if `Loaded` is `False`. Substitute the **actual active profile path**.

4. From an **elevated** PowerShell session, rename the whole cache folder, including its ESENT database and transaction logs, and optionally the associated lock file:

   ```powershell
   $profile = 'C:\Users\example.user.DOMAIN'  # CHANGE THIS
   $base    = Join-Path $profile 'AppData\Local\Microsoft\Windows'

   Rename-Item -LiteralPath (Join-Path $base 'WebCache') -NewName 'WebCache.old'

   $lock = Join-Path $base 'WebCacheLock.dat'
   if (Test-Path -LiteralPath $lock) {
       Rename-Item -LiteralPath $lock -NewName 'WebCacheLock.dat.old'
   }
   ```

   If the rename fails because the folder is still in use, **stop and investigate**; do not force deletion. If `WebCache.old` already exists, choose a different backup name.

5. Sign the affected user in again. Windows recreates the cache as required.
6. Retest the symptoms, then check whether **new** ESENT `WebCacheLocal` recovery errors have ceased.

**Trade-off:** Rebuilding this legacy WinINet cache may remove cached web data and associated cookies/session state. Keep `WebCache.old` temporarily for rollback. Renaming only `WebCacheV01.dat` while leaving its existing transaction logs is not the procedure used here.

## Actual outcome

After the cache folder and lock file were renamed with the user signed out, and the user logged back in:

- **Chrome downloads completed normally.**
- **Windows Settings opened immediately.**
- **LAN settings opened immediately.**
- **Outlook opened and started downloading email.**

No full profile rebuild or Windows reinstall was required.

## Extra diagnostic evidence (not necessary for the first-pass fix)

For anyone facing the same symptoms *without* obvious ESENT log entries:

- ProcMon showed Chrome downloading and renaming files successfully before the browser UI remained stuck.
- Chrome's child process with `--utility-sub-type=quarantine.mojom.Quarantine` remained blocked.
- Process Explorer showed that process in `urlmon.dll` / `wininet.dll`, in a synchronous RPC/ALPC wait.
- Windows **Analyze wait chain** identified `taskhostw.exe` as a dependency.
- A thread stack from `taskhostw.exe` showed `wininet.dll` calling into **ESENT.dll** database initialization/recovery (`JetInit4W`, `JetAttachDatabaseW`).
- `\Microsoft\Windows\Wininet\CacheTask` was running, but stopping that scheduled task **did not** unblock downloads. The successful intervention was resetting the affected user's WebCache while signed out.

**In short:** When a **Transwiz-migrated Windows 11 profile** produces several unrelated-looking application hangs, **check ESENT / `WebCacheLocal` first**. If it shows repeated database recovery failures, a backed-up reset of that user's **`AppData\Local\Microsoft\Windows\WebCache`** may resolve them.

*Case report from one successful repair; correlation with Transwiz does not establish a universal product defect.*
