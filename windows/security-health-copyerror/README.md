# Windows Security won't open: zeroed Security Health binaries and `CopyError`

| | |
|---|---|
| **Status** | Fixed |
| **OS** | Windows 11 26H2, domain-joined |
| **Component** | Windows Security platform (KB5007651) |
| **Environment** | Third-party EDR active (Cisco Secure Endpoint), Defender in passive mode |
| **Fixed** | October 2026 |

## Summary

The Windows Security app would not open. Two binaries in the Windows Security platform folder had been zeroed out during a platform update months earlier. The registry still pointed the platform at that broken folder, and every later platform update failed with `CopyError`.

DISM, SFC and running the platform installer by hand did not help. What fixed it was pointing the platform's `CoreLocation` back to System32 as SYSTEM, removing the leftover installer, and letting Windows Update reinstall KB5007651.

## Symptoms

- Windows Security (Windows-Sicherheit) does not open.
- At login, an error dialog appears:
  ```
  SecurityHealthSystray: SecurityHealthSystray.exe - Ungültiges Bild
  \\?\C:\Windows\System32\SecurityHealth\10.0.29554.1001-0\SecurityHealthSSO.dll ist entweder
  nicht für die Ausführung unter Windows vorgesehen oder enthält einen Fehler. [...]
  Fehlerstatus 0xc000012f.
  ```
  In English this is the "Bad Image" dialog. `0xc000012f` means the file is not a valid executable image.
- These services show as stopped: `SecurityHealthService`, `WdNisSvc`, `WdNisDrv`, `WdBoot`.
- Restarting them gives "wurde gestartet und dann angehalten" (started and then stopped). Starting them from an admin PowerShell fails as well.

## Red herrings

Most of the stopped services looked like the problem but were behaving normally.

- **`WdBoot`** is Defender's early-launch (ELAM) boot driver. It loads during boot and unloads afterwards, so "stopped" is its normal state.
- **`WdNisSvc` and `WdNisDrv`** are Defender's network inspection components. They only run when Defender is the active antivirus. Here Defender was deliberately passive because a third-party EDR was registered.
- **`WinDefend`** was running the whole time. It is a protected service, so admins cannot start or stop it by hand anyway.
- **No policy was involved.** Both Defender policy keys were empty and Tamper Protection was on.

```powershell
Get-Service WinDefend, SecurityHealthService, wscsvc, WdNisSvc, Sense | ft Name, Status, StartType
Get-MpComputerStatus | fl AMRunningMode, AntivirusEnabled, RealTimeProtectionEnabled, IsTamperProtected
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender" -ErrorAction SilentlyContinue
```

Result: `WinDefend` running, `AMRunningMode : Passive Mode`, `IsTamperProtected : True`, no policy values.

The Windows Security app package (`Microsoft.SecHealthUI`) was also fine (`Status : Ok`). The app failed because its backend, `SecurityHealthService`, could not load its components.

## Diagnosis

### 1. The file from the error dialog is zeroed

```powershell
$f = "C:\Windows\System32\SecurityHealth\10.0.29554.1001-0\SecurityHealthSSO.dll"
Get-Item $f | select Length, LastWriteTime
[IO.File]::ReadAllBytes($f)[0..1]
Get-AuthenticodeSignature $f | fl Status, StatusMessage
```

| Check | Result | Expected |
|---|---|---|
| Size | 1,418,728 bytes | normal size |
| LastWriteTime | 26.05.2026 09:25 | |
| First two bytes | `0 0` | `77 90` ("MZ") |
| Signature | `UnknownError` | `Valid` |

The file had its full size but contained only zeros. Space was allocated, but the data was never written.

### 2. Scope: two files in one folder

```powershell
$paths = "C:\Windows\System32\SecurityHealth", "C:\ProgramData\Microsoft\Windows Defender\Platform"
Get-ChildItem $paths -Recurse -Include *.dll,*.exe -ErrorAction SilentlyContinue |
  Get-AuthenticodeSignature | ? Status -ne 'Valid' | ft Path, Status -AutoSize
```

Only `SecurityHealthSSO.dll` and `SecurityHealthHost.exe` in `10.0.29554.1001-0` were broken. The Defender platform folder was clean.

### 3. Disk is healthy

```powershell
Get-PhysicalDisk | ft FriendlyName, MediaType, HealthStatus, OperationalStatus
chkdsk C: /scan
```

Disk healthy, chkdsk found no problems. That ruled out ongoing hardware failure.

### 4. Registry shows a stuck platform update

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows Security Health\Platform" | fl Registered, CoreLocation
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows Security Health\Updates"
```

| Value | Broken machine | Healthy machine |
|---|---|---|
| `Platform\Registered` | `0` | `1` |
| `Platform\CoreLocation` | `...\SecurityHealth\10.0.29554.1001-0` | `...\SecurityHealth\10.0.29628.1000-0` |
| `Updates\wu` | `CopyError` | `10.0.29628.1000` |

`CoreLocation` tells Windows which folder to load the Security Health components from. On the broken machine it still pointed at the folder with the zeroed files. `wu = CopyError` matches a known failure state documented by Microsoft's Japanese security support team (see references).

There was also a `SecurityHealthSetup.exe` (version 10.0.29628.1000, valid Microsoft signature) in `C:\Windows\System32\SecurityHealth`, written the same day by Windows Update.

## Root cause

The Windows Security platform update to 10.0.29554.1001 on 26.05.2026 left two binaries zeroed. The pattern (correct size, zero content, healthy disk) points to an interrupted write, for example a hard shutdown shortly after the update. The exact trigger is unknown. The EDR also registered a new version that same morning. That is only a correlation.

Because `CoreLocation` still pointed at the broken folder, `SecurityHealthService` kept loading the zeroed files and exited immediately. Later platform updates failed with `CopyError` and never replaced the broken version.

## What didn't work

- **DISM `/RestoreHealth` followed by `sfc /scannow`.** Both completed successfully, but the files stayed zeroed. SFC does not appear to service the versioned `SecurityHealth` folder.
- **Running `SecurityHealthSetup.exe` manually as admin.** Exit code 0, nothing changed. Most likely the installer considered the platform already installed.
- **Copying the files from a healthy machine.** The only healthy machine available was already on 10.0.29628.1000. Mixing files from two versions into one folder was not attempted.

## Fix

Based on the Microsoft Japan CSS article on KB5007651 being reoffered (see references). Their scenario is an update loop, not zeroed files, but the mechanism is the same.

**Requirements:** local admin, [PsExec](https://learn.microsoft.com/en-us/sysinternals/downloads/psexec) for a SYSTEM shell.

> **Heads-up:** EDR products often flag PsExec because it is a common lateral-movement tool. Expect an alert and tell your security team beforehand.

### 1. Back up the registry key

```powershell
New-Item -ItemType Directory C:\Temp -Force | Out-Null
reg export "HKLM\SOFTWARE\Microsoft\Windows Security Health" C:\Temp\SecurityHealth-backup.reg /y
```

### 2. Check that the System32 fallback is intact

After the reset, Windows loads the built-in copies from System32. All three must be `Valid`. If not, stop here.

```powershell
Get-AuthenticodeSignature C:\Windows\System32\SecurityHealthService.exe, C:\Windows\System32\SecurityHealthHost.exe, C:\Windows\System32\SecurityHealthSSO.dll | ft Path, Status -AutoSize
```

### 3. Open a SYSTEM PowerShell

Extract `PsExec64.exe` to `C:\Temp`, then from an admin console:

```powershell
C:\Temp\PsExec64.exe -s -i -accepteula powershell.exe
```

`whoami` in the new window should return `nt authority\system` (`nt-autorität\system` on German systems).

### 4. Remove the leftover installer and reset `CoreLocation`

In the SYSTEM window:

```powershell
Move-Item C:\Windows\System32\SecurityHealth\SecurityHealthSetup.exe C:\Temp\ -Force
$reg = "HKLM:\SOFTWARE\Microsoft\Windows Security Health\Platform"
Set-ItemProperty $reg -Name CoreLocation -Value '\\?\C:\Windows\System32'
Get-ItemProperty $reg | fl Registered, CoreLocation
```

`Registered` was already `0` in this case. If it is `1`, also run `Set-ItemProperty $reg -Name Registered -Value 0`, as the Microsoft article describes.

Close the SYSTEM window and delete `C:\Temp\PsExec64.exe`.

### 5. Reboot

The Bad Image error at login disappears and Windows Security opens again, running the built-in version.

### 6. Install KB5007651 through Windows Update, then reboot

If the device gets updates through WSUS, KB5007651 needs to be approved there.

## Verification

```powershell
Get-ChildItem C:\Windows\System32\SecurityHealth
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows Security Health\Platform" | fl Registered, CoreLocation
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows Security Health\Updates"
```

Expected after the fix, and confirmed in this case:

- New folder `10.0.29628.1000-0`
- `Registered : 1`
- `CoreLocation : \\?\C:\Windows\System32\SecurityHealth\10.0.29628.1000-0`
- `wu : 10.0.29628.1000`
- Windows Security opens normally

The old `10.0.29554.1001-0` folder may stay behind. It is harmless once nothing references it.

## Rollback

```powershell
reg import C:\Temp\SecurityHealth-backup.reg
Move-Item C:\Temp\SecurityHealthSetup.exe C:\Windows\System32\SecurityHealth\ -Force   # as SYSTEM
```

## Side notes

- **Stale antivirus registrations.** `root/SecurityCenter2` listed several old versions of the EDR and a previously removed antivirus product, each with its own entry. Only the current EDR was actually installed. These leftovers are cosmetic and unrelated to this bug. Clean them up with the vendor's removal tools, not by editing WMI.
- **Other machines.** If one machine got zeroed files from a platform update, others updated at the same time may have them too. The signature check from diagnosis step 2 is a quick way to find out.

## References

- Microsoft Japan CSS Security Support Blog: [KB5007651 が何度も再オファーされる事象へのトラブルシューティング](https://jp-sec.github.io/blog/Endpoint-MDAV-2025-04-16/) (troubleshooting KB5007651 being reoffered repeatedly)
- Microsoft Support: [KB5007651, Windows Security app update](https://support.microsoft.com/help/5007651)
- Sysinternals: [PsExec](https://learn.microsoft.com/en-us/sysinternals/downloads/psexec)
