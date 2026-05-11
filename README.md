# Dell Bloatware Cleanup Guide

**Author:** Claude Code Agent
**Date:** 2026-05-11
**Applies to:** Dell machines with SupportAssist, TechHub, Dell Data Vault, SmartByte/Rivet Networks, and related Dell/Rivet software

---

## Overview

The official Dell SupportAssist uninstaller does not fully remove all Dell software. Several components survive and auto-start at boot. Additionally, Dell machines with Killer/Ethernet NICs often ship with **SmartByte/Rivet Networks** — a network optimization suite that is bloatware for most users.

This guide covers full, safe removal in the correct order.

**What was removed on this machine (DESKTOP-JLD47LG, 2026-05-11):**
- ~1.5 GB total freed (Registry + ProgramData + AppData + Revo deep scan)
- SmartByte Drivers and Services, all Rivet Networks services, SmbCoSvc kernel driver
- All Dell SupportAssist/TechHub registry ghosts
- C:\ProgramData\SupportAssistDbBackup\ (516 MB), C:\ProgramData\RivetNetworks\ ImageCache (2.3 MB)

---

## What to Keep (Dell Legitimate Software)

| Software | Reason to Keep |
|---|---|
| **Dell Update for Windows Universal** | Required for BIOS/firmware/driver updates |
| **Dell Update Service** (`C:\Program Files (x86)\Dell\UpdateService\`) | Update engine for BIOS, drivers, and Dell firmware |
| **DellInstrumentation.sys** | Kernel driver for thermal/power management — part of Dell driver suite from `oem89.inf`. Currently running cleanly. Safe to leave if no issues. |

### What NOT to Keep (Bloatware)
- **Dell Digital Delivery Services** — Separate application from Dell Update, used for pre-installed Dell software delivery. NOT needed for BIOS updates. Safe to remove.
- **SmartByte Drivers and Services (Rivet Networks)** — Network optimization/gaming traffic shaper from Killer Networking. Bundled with Dell XPS/Alienware/gaming laptops. Consumer bloatware — safe to remove even if you use Killer NIC hardware (the NIC works without SmartByte).
- `C:\Program Files\Rivet Networks\` — SmartByte application files
- `C:\Windows\System32\DRIVERS\SmbCo10X64.sys` — SmartByte kernel driver

---

## Phase 1 — Discover: Find Everything Dell + SmartByte

### 1.1 List all Dell/Rivet services

```powershell
Get-Service | Where-Object {
    $_.Name -like '*Dell*' -or $_.Name -like '*SupportAssist*' -or $_.Name -like '*DDV*' -or
    $_.Name -like '*Fusion*' -or $_.Name -like '*SmartByte*' -or $_.Name -like '*Rivet*' -or
    $_.Name -like '*RAPS*' -or $_.Name -like '*RNDBWM*' -or $_.Name -like '*SmbCo*'
} | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

### 1.2 List all Dell/SmartByte processes currently running

```powershell
Get-Process | Where-Object {
    $_.ProcessName -like '*Dell*' -or $_.ProcessName -like '*SupportAssist*' -or
    $_.ProcessName -like '*SmartByte*' -or $_.ProcessName -like '*Rivet*'
} | Select-Object ProcessName, Id, @{
    N='RAM_MB';E={[math]::Round($_.WorkingSet64/1MB,1)}
} | Format-Table -AutoSize
```

Or via CMD:

```cmd
tasklist | findstr /i /c:"dell" /c:"supportassist" /c:"ddv" /c:"vault" /c:"smartbyte" /c:"rivet" /c:"smbco"
```

### 1.3 Find all Dell/Rivet programs in Add/Remove Programs (registry)

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
                'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*',
                'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue |
    Where-Object { $_.Publisher -like '*Dell*' -or $_.Publisher -like '*Rivet*' -or
                   $_.DisplayName -like '*Dell*' -or $_.DisplayName -like '*SmartByte*' } |
    Select-Object PSChildName, DisplayName, Publisher, UninstallString, @{
        N='Arch';E={ if ($_.PSPath -like '*WOW6432Node*') { 'x86' } else { 'x64' } }
    } | Format-Table -AutoSize
```

**Important:** Check `HKLM:\SOFTWARE\Classes\Installer\Products\` — Revo Uninstaller scans this but Windows Add/Remove does not. Ghost entries for SupportAssist, SmartByte, Mobile Connect, Digital Delivery often live here with no uninstall string.

### 1.4 Find Dell/SmartByte program file folders

```powershell
Get-ChildItem 'C:\Program Files\Dell' -ErrorAction SilentlyContinue | Select-Object Name, FullName
Get-ChildItem 'C:\Program Files (x86)\Dell' -ErrorAction SilentlyContinue | Select-Object Name, FullName
Get-ChildItem 'C:\Program Files\Rivet Networks' -ErrorAction SilentlyContinue | Select-Object Name, FullName
```

### 1.5 Find all ProgramData/AppData Dell leftovers

Revo Uninstaller finds these when scanning beyond standard locations:

```powershell
# ProgramData (shared app data)
Get-ChildItem 'C:\ProgramData' -Force -ErrorAction SilentlyContinue |
    Where-Object { $_.Name -match 'Dell|Rivet|SupportAssist|SmartByte' } |
    ForEach-Object { Write-Output $_.FullName }

# AppData Local
Get-ChildItem 'C:\Users\*\AppData\Local\Dell*','C:\Users\*\AppData\Local\Rivet*' -Force -ErrorAction SilentlyContinue |
    ForEach-Object { Write-Output $_.FullName }

# AppData Roaming
Get-ChildItem 'C:\Users\*\AppData\Roaming\Dell*','C:\Users\*\AppData\Roaming\Rivet*' -Force -ErrorAction SilentlyContinue |
    ForEach-Object { Write-Output $_.FullName }
```

---

## Phase 2 — Stop and Disable Services FIRST

**Critical rule:** Stop services before disabling them. The order matters on a live system.

For each service found in Phase 1:

```powershell
# Run elevated (RunAs Administrator)
sc stop <ServiceName>
sc config <ServiceName> start= disabled
```

### DDV Services — Often Missed by SupportAssist Uninstall

These three DDV services are not removed by the SupportAssist uninstaller and will auto-start at boot:

```powershell
sc stop DDVCollectorSvcApi
sc config DDVCollectorSvcApi start= disabled

sc stop DDVDataCollector
sc config DDVDataCollector start= disabled

sc stop DDVRulesProcessor
sc config DDVRulesProcessor start= disabled
```

**How to identify DDV if hidden:** They show in Task Manager as:
- Dell Data Vault Rules Processor (`DDVRulesProcessor.exe`)
- Dell Data Vault Collector Service API (`DDVCollectorSvcApi.exe`)
- DDVDataCollector (`DDVDataCollector.exe`)

They load from: `C:\Program Files\Dell\DellDataVault\`

### SmartByte/Rivet Services

These services should all be stopped + disabled:

```powershell
sc stop "SmartByte Network Service x64"
sc config "SmartByte Network Service x64" start= disabled

sc stop "SmartByte Analytics Service"
sc config "SmartByte Analytics Service" start= disabled

sc stop RAPSService
sc config RAPSService start= disabled

sc stop RNDBWM
sc config RNDBWM start= disabled

sc stop SmbCoSvc
sc config SmbCoSvc start= disabled
```

### Verify services won't restart

After stopping and disabling, reboot and check. All should be `STOPPED` and `START_TYPE: 4` (Disabled).

---

## Phase 3 — Remove Registry Entries

**Only do this AFTER stopping and disabling services.**

### 3.1 Remove Add/Remove Programs registry entries

Standard Uninstall keys:
```cmd
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{<GUID>}" /f
```

### 3.2 Remove Classes\Installer\Products ghost entries

**These are the entries Revo Uninstaller finds that Windows Add/Remove misses.** GUID pattern:

```powershell
# Run elevated
Remove-Item -Path "HKLM:\SOFTWARE\Classes\Installer\Products\<GUID>" -Recurse -Force
```

Common ghosts found on Dell machines:

| Product Name | GUID (partial) |
|---|---|
| Dell SupportAssist | 61A9D37AB22CC2A4D... |
| Dell SupportAssist Remediation | 672DFE3DA76FDC548... |
| Dell Digital Delivery Services | 5F5434B775B861741... |
| Dell Mobile Connect Drivers | F2B1074433D795F43... |
| SmartByte Drivers and Services | D3DADC0A9230E3E4D... |
| Dell SupportAssist OS Recovery Plugin | 4CD5FBE6B0AF29649... |

**Note:** To enumerate all Dell/SmartByte ghosts in Classes\Installer\Products:

```powershell
Get-ChildItem 'HKLM:\SOFTWARE\Classes\Installer\Products' -Force | ForEach-Object {
    $props = Get-ItemProperty $_.PSPath
    if ($props.ProductName -match 'Dell|Support|SmartByte|Rivet|Mobile|Digital|Delivery|TechHub') {
        Write-Output "$($_.PSChildName) : $($props.ProductName)"
    }
}
```

### 3.3 Remove HKLM:\SOFTWARE\Dell bloat subkeys

After removing SupportAssist/TechHub services, clean the HKLM:\SOFTWARE\Dell tree:

```powershell
# Run elevated - delete all bloat subkeys, keep UpdateService and UpdatePlugins
$bloatKeys = @(
    'SupportAssistAgent',
    'SupportAssistHardwareDiagnosticsSubAgent',
    'SupportAssistOsRecoverySubAgent',
    'SupportAssistServiceSubAgent',
    'SupportAssistSoftwareDiagnosticsSubAgent',
    'Dell TechHub',
    'Dell Digital Delivery',
    'Dell.DataVault.DataCollector.SubAgent',
    'DTP.Analytics.SubAgent',
    'DTP.Commodity.SubAgent',
    'DTP.DataManager.SubAgent',
    'DTP.Diagnostics.SubAgent',
    'DTP.Instrumentation.SubAgent',
    'DTP.Transmission',
    'DTPRefCountAdjustment',
    'ManageableUpdatePackage',
    'SARemediation',
    'Bradbury API Sub Agent',
    'CoreServices.Client',
    'DCFShared',
    'Fusion',
    'MUP',
    'Notification Manager'
)

foreach ($key in $bloatKeys) {
    Remove-Item "HKLM:\SOFTWARE\Dell\$key" -Recurse -Force -ErrorAction SilentlyContinue
}
```

**Keep:** `HKLM:\SOFTWARE\Dell\UpdateService` and `HKLM:\SOFTWARE\Dell\UpdatePlugins` — Dell Update engine.

### 3.4 Remove HKLM:\SYSTEM\CurrentControlSet\Services entries for SmartByte/Rivet

```powershell
# Run elevated
Remove-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\SmartByte Analytics Service" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\SmartByte Network Service x64" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\RAPSService" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\RNDBWM" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\SmbCoSvc" -Recurse -Force -ErrorAction SilentlyContinue
```

### 3.5 Check for Dell/SmartByte scheduled tasks

```powershell
Get-ScheduledTask | Where-Object {
    $_.TaskName -match 'Dell|SmartByte|SupportAssist|TechHub|DDV'
} | Select-Object TaskName, State | Format-Table -AutoSize
```

Delete any found:

```powershell
Unregister-ScheduledTask -TaskName "<TaskName>" -Confirm:$false
```

### 3.6 Check Run/RunOnce keys

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\*',
                'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\*' -ErrorAction SilentlyContinue |
    Where-Object { $_ -match 'Dell|SmartByte|Rivet' } | Format-List
```

---

## Phase 4 — Remove Program Files and Folders

**Only do this AFTER services are stopped and disabled.**

### 4.1 Standard Dell folders to remove

| Path | What it is |
|---|---|
| `C:\Program Files\Dell\DellDataVault` | DDV — almost always present after SupportAssist uninstall |
| `C:\Program Files\Dell\DellMobileConnectDrivers` | Dell Mobile Connect — bloatware |
| `C:\Program Files\Dell\DTP` | Diagnostic/SupportAssist related |
| `C:\Program Files\Dell\Fusion` | Dell Fusion — bloatware |
| `C:\Program Files\Dell\Plugins` | TechHub/SupportAssist plugins |
| `C:\Program Files\Dell\SARemediation` | SupportAssist Remediation |
| `C:\Program Files\Dell\SupportAssistAgent` | Stub from incomplete uninstall |
| `C:\Program Files\Dell\TechHub` | Dell TechHub — bloatware |
| `C:\Program Files\Dell\Update` | Empty 64-bit stub — safe to remove |
| `C:\Program Files (x86)\Dell Digital Delivery Services` | **Separate bloatware app** — NOT Dell Update |

```cmd
rmdir /s /q "C:\Program Files\Dell\DellDataVault"
rmdir /s /q "C:\Program Files\Dell\DellMobileConnectDrivers"
rmdir /s /q "C:\Program Files\Dell\DTP"
rmdir /s /q "C:\Program Files\Dell\Fusion"
rmdir /s /q "C:\Program Files\Dell\Plugins"
rmdir /s /q "C:\Program Files\Dell\SARemediation"
rmdir /s /q "C:\Program Files\Dell\SupportAssistAgent"
rmdir /s /q "C:\Program Files\Dell\TechHub"
rmdir /s /q "C:\Program Files\Dell\Update"
rmdir /s /q "C:\Program Files (x86)\Dell Digital Delivery Services"
```

### 4.2 SmartByte / Rivet Networks folders

```cmd
rmdir /s /q "C:\Program Files\Rivet Networks"
```

### 4.3 SmartByte kernel driver files

```cmd
del /f "C:\Windows\System32\DRIVERS\SmbCo10X64.sys"
del /f "C:\Windows\System32\DRIVERS\SmbCoX64w10.inf"
del /f "C:\Windows\System32\DRIVERS\smbcox64w10.cat"
```

### 4.4 ProgramData Dell/Rivet/AppData Dell leftovers

After registry is clean, remove remaining data folders:

```cmd
# ProgramData (can be large - Revo often finds 500MB+ here)
rmdir /s /q "C:\ProgramData\Dell Inc"
rmdir /s /q "C:\ProgramData\RivetNetworks"
rmdir /s /q "C:\ProgramData\SupportAssist"
rmdir /s /q "C:\ProgramData\SupportAssistDbBackup"

# AppData Local Dell (SmartByte telemetry cache)
rmdir /s /q "C:\Users\<Username>\AppData\Local\Dell"
```

### 4.5 x86 Dell folder — be selective

`C:\Program Files (x86)\Dell\UpdateService\` — **KEEP this** if you want Dell BIOS/driver updates.

Everything else in the x86 Dell folder can be removed if you know Dell Update is the only thing there.

---

## Phase 5 — Reboot and Verify

After all removals, **reboot the machine** and run:

```cmd
tasklist | findstr /i /c:"dell" /c:"supportassist" /c:"ddv" /c:"smartbyte" /c:"rivet" /c:"smbco" /c:"fusion" /c:"techhub"
```

Should return *empty*.

Then check services:

```powershell
Get-Service | Where-Object {
    $_.Name -like '*Dell*' -or $_.Name -like '*SupportAssist*' -or
    $_.Name -like '*DDV*' -or $_.Name -like '*SmartByte*' -or
    $_.Name -like '*Rivet*' -or $_.Name -like '*Fusion*'
}
```

All should be `Stopped` except `DellInstrumentation` which is a thermal driver (keep it).

Then check registry — only `Dell Update for Windows Universal` should remain:

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue |
    Where-Object { $_.Publisher -like '*Dell*' -and $_.DisplayName -notlike '*Update*' } |
    Select-Object DisplayName
```

Should return empty.

Finally, run **Revo Uninstaller** to do a final deep scan — it consistently finds leftover files in ProgramData and AppData that scripts miss.

---

## Phase 6 — Optional: Remove Dell Update Completely

If you do not need BIOS updates and want a fully clean machine:

```powershell
# Stop the Update Service
sc stop "Dell Update Service"   # (find actual service name first via Get-Service)

# Remove the x86 Dell folder
Remove-Item -Path "C:\Program Files (x86)\Dell" -Recurse -Force
```

---

## Troubleshooting

### Access denied on `sc stop` or `sc config`

**Cause:** Not running as Administrator.

**Fix:** Use `Start-Process powershell -Verb RunAs -Wait` pattern or open an elevated PowerShell prompt (Run as Administrator).

### A service restarts immediately after I stop it

**Cause:** The service is set to `Automatic` (StartType=2) instead of `Disabled`. Stop it first, then immediately run `sc config <name> start= disabled`.

### Revo Uninstaller shows entries that aren't in the registry

**Cause:** Revo scans `HKLM:\SOFTWARE\Classes\Installer\Products\` which Windows Add/Remove doesn't show. Also scans `HKLM:\SOFTWARE\Dell` registry tree directly.

**Fix:** Delete the entries from both locations. See Phase 3.2 and 3.3.

### After reboot, Dell software reappeared

**Cause:** Either the MSI uninstaller didn't fully remove dependent services, or Dell Update reinstalled components.

**Fix:** The DDV services are the most common culprits — they are separate from SupportAssist and survive its uninstall. Use Phase 2 to stop/disable them, then Phase 3 to remove registry entries, then Phase 4 to remove folders.

### Registry entries remain for deleted services

**Cause:** Previous cleanup attempts deleted the service entries but left orphaned registry keys pointing to non-existent executables.

**Fix:** Use `sc delete <ServiceName>` to fully remove service entries from the registry. Only do this after confirming the service executable no longer exists at the path referenced in the registry entry.

### SmartByte uninstall string shows `/I` instead of `/X`

**Cause:** The registry entry is the original MSI installer GUID, not an uninstall entry. This happens when SmartByte was installed via MSI but the uninstaller never ran (or the entry was never created properly).

**Fix:** The `/I` means "install" — not uninstallable via MSI. You must delete the registry entry manually and remove the folder manually. See Phase 3.2 and Phase 4.2.

### SmbCoSvc kernel driver blocks folder deletion

**Cause:** Driver is still loaded in memory even after service stop.

**Fix:** Reboot first (before deleting the folder), then delete after reboot. The driver unloads after reboot.

### Revo shows ~1GB in C:\ProgramData\Dell\ but I can't find it

**Cause:** ProgramData folders may be hidden or require admin permissions to view/delete.

**Fix:** Open File Explorer → View → Show hidden files. Use Revo to delete or run CMD as Administrator to delete.

---

## Key Lessons

1. **SupportAssist MSI uninstaller does NOT remove DDV services.** Always check for DDV specifically.
2. **DDV services auto-start (StartType=2) from a separate `C:\Program Files\Dell\DellDataVault\` folder.**
3. **Multiple registry uninstall stubs** survive in both `HKLM:\Uninstall` and `HKLM:\WOW6432Node\Uninstall` hives.
4. **Revo Uninstaller scans locations Windows doesn't show** — Classes\Installer\Products, HKLM:\SOFTWARE\Dell, ProgramData, AppData. Always run Revo after manual cleanup to catch leftovers.
5. **Both x64 and x86 Dell folders exist** (`C:\Program Files\Dell` and `C:\Program Files (x86)\Dell`). The x86 folder holds the real Dell Update engine.
6. **Dell Digital Delivery Services is separate from Dell Update** — not needed for BIOS updates, safe to remove.
7. **SmartByte/Rivet Networks is Killer Networking's bloatware** — not needed for the NIC to work, safe to remove. SmbCoSvc kernel driver must be unloaded before deleting folder.
8. **ProgramData can hold 500MB+ of Dell/SmartByte cache** — always scan and clean it.
9. **The order should always be:** Identify → Stop → Disable → Delete registry → Remove files → Reboot → Verify → Run Revo.

---

## Quick Reference: Commands to Run

```bash
# 1. DISCOVER
Get-Service | Where-Object { $_.Name -like '*Dell*' -or $_.Name -like '*SupportAssist*' -or $_.Name -like '*DDV*' -or $_.Name -like '*SmartByte*' -or $_.Name -like '*Rivet*' }
tasklist | findstr /i /c:"dell" /c:"supportassist" /c:"ddv" /c:"smartbyte" /c:"rivet" /c:"fusion"

# 2. STOP + DISABLE (run each service name)
sc stop <ServiceName>
sc config <ServiceName> start= disabled

# 3. DELETE REGISTRY (use GUID from Phase 1.3)
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{<GUID>}" /f
Remove-Item -Path "HKLM:\SOFTWARE\Classes\Installer\Products\<GUID>" -Recurse -Force

# 4. REMOVE FOLDERS
rmdir /s /q "C:\Program Files\Dell\DellDataVault"
rmdir /s /q "C:\Program Files\Rivet Networks"
rmdir /s /q "C:\ProgramData\Dell Inc"
rmdir /s /q "C:\ProgramData\RivetNetworks"
rmdir /s /q "C:\ProgramData\SupportAssistDbBackup"
del /f "C:\Windows\System32\DRIVERS\SmbCo10X64.sys"

# 5. REBOOT AND VERIFY
tasklist | findstr /i /c:"dell" /c:"supportassist" /c:"ddv" /c:"smartbyte" /c:"rivet"
# Should be empty

# 6. RUN REVO UNINSTALLER for deep scan
```

---

## Summary

This guide provides a systematic six-phase approach to completely remove Dell bloatware from Windows systems — including SmartByte/Rivet Networks bloatware found on Dell machines with Killer NICs. The key insights are:

1. Dell's own uninstaller often leaves behind the Dell Data Vault (DDV) services, which continue to run at startup
2. SmartByte/Rivet Networks is a separate network optimization bloatware that survives its own uninstaller
3. Revo Uninstaller finds 500MB–1GB+ of leftovers in ProgramData that scripts miss
4. The recommended order is: discover all Dell/SmartByte components, stop and disable services, remove registry entries (including Classes\Installer\Products), delete program folders, clean ProgramData, reboot, verify, then run Revo for final sweep

Users should retain Dell Update if they need BIOS or driver updates, but can safely remove Dell Digital Delivery Services, SmartByte/Rivet Networks, and other non-essential Dell/Rivet software.