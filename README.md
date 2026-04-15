# Dell Bloatware Cleanup Guide

**Author:** Claude Code Agent
**Date:** 2026-04-15
**Applies to:** Dell machines with SupportAssist, TechHub, Dell Data Vault, and related Dell software

---

## Overview

The official Dell SupportAssist uninstaller (`MSI` or `Uninstall.exe`) does **not fully remove all Dell software. Several components survive and auto-start at boot:**

- **Dell Data Vault (DDV)** — Three services that the SupportAssist uninstall never touches
- **Dell Update (x86)** — Separate from the 64-bit Dell Update; contains the actual update engine
- **Various stub folders and registry entries** — Left behind after incomplete uninstalls

This guide covers a **full, safe removal** in the correct order.

---

## What to Keep (Dell Legitimate Software)

| Software | Reason to Keep |
|---|---|
| **Dell Update for Windows Universal** | Required for BIOS/firmware updates |
| **Dell Update Service** (`C:\Program Files (x86)\Dell\UpdateService`) | Update engine for BIOS, drivers, and Dell firmware |

If you are okay with not getting Dell BIOS updates, you can remove the Update Service too. This guide removes all bloatware **but preserves the Update components**.

---

## Phase 1 — Discovery: Find Everything Dell

Before removing anything, get a complete picture.

### 1.1 List all Dell services

```powershell
Get-Service | Where-Object { $_.Name -like '*Dell*' -or $_.Name -like '*SupportAssist*' -or $_.Name -like '*DDV*' -or $_.Name -like '*DTP*' -or $_.Name -like '*Fusion*' } | `
    Select-Object Name, Status, StartType | Format-Table -AutoSize
```

### 1.2 List all Dell processes currently running

```powershell
Get-Process | Where-Object { $_.Name -like '*Dell*' -or $_.Name -like '*SupportAssist*' -or $_.Name -like '*DDV*' } | `
    Select-Object Name, Id, @{N='RAM_MB';E={[math]::Round($_.WorkingSet64/1MB,1)}} | Format-Table -AutoSize
```

Or via CMD:

```cmd
tasklist | findstr /i /c:"dell" /c:"supportassist" /c:"ddv" /c:"vault"
```

### 1.3 Find all Dell programs in Add/Remove Programs (registry)

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*', `
                 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue | `
    Where-Object { $_.Publisher -like '*Dell*' -or $_.DisplayName -like '*Dell*' -or $_.DisplayName -like '*SupportAssist*' } | `
    Select-Object PSChildName, DisplayName, Publisher, UninstallString, @{N='Hive';E={
        if ($_.PSPath -like '*WOW6432Node*') { 'x86' } else { 'x64' }
    }} | Format-Table -AutoSize
```

**Important:** Note the `PSChildName` (registry subkey name) for each entry — you will need these for removal.

### 1.4 Find Dell program file folders

```powershell
Get-ChildItem 'C:\Program Files\Dell' -ErrorAction SilentlyContinue | Select-Object Name, FullName
Get-ChildItem 'C:\Program Files (x86)\Dell' -ErrorAction SilentlyContinue | Select-Object Name, FullName
```

---

## Phase 2 — Stop and Disable Services FIRST

**Critical rule:** Stop services **before** disabling them. The order matters on a live system.

For each Dell service found in Phase 1:

```powershell
# Run elevated (RunAs Administrator)
sc stop <ServiceName>
sc config <ServiceName> start= disabled
```

### DDV Services — Often Missed

These three DDV services are **not removed by the SupportAssist uninstaller** and will auto-start at boot:

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

### Verify services won't restart

After stopping and disabling, reboot and check:

```cmd
sc query DellClientManagementService
sc query DDVCollectorSvcApi
```

State should be `STOPPED` and `START_TYPE: 4` (Disabled).

---

## Phase 3 — Remove Registry Entries

**Only do this AFTER stopping and disabling services.**

### 3.1 Remove Add/Remove Programs registry entries

Use the `PSChildName` from Phase 1.3:

```cmd
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{<PSChildName>}" /f
reg delete "HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\{<PSChildName>}" /f
```

### 3.2 Check for Dell scheduled tasks

```powershell
Get-ScheduledTask | Where-Object { $_.TaskName -like '*Dell*' -or $_.TaskName -like '*SupportAssist*' -or $_.TaskName -like '*DDV*' } | `
    Select-Object TaskName, State | Format-Table -AutoSize
```

Disable or delete any found:

```powershell
Disable-ScheduledTask -TaskName "<TaskName>"
Unregister-ScheduledTask -TaskName "<TaskName>" -Confirm:$false
```

### 3.3 Check Run/RunOnce keys

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\*', 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\*' -ErrorAction SilentlyContinue | `
    Where-Object { $_ -like '*Dell*' } | Format-List
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
```

### 4.2 x86 Dell folder — be selective

`C:\Program Files (x86)\Dell\UpdateService` — **Keep this** if you want Dell BIOS updates.

Everything else in the x86 Dell folder (e.g., Dell Digital Delivery) can be removed.

---

## Phase 5 — Reboot and Verify

After all removals, **reboot the machine** and run:

```cmd
tasklist | findstr /i /c:"dell" /c:"supportassist" /c:"ddv" /c:"vault"
```

Should return **empty**.

Then check services:

```powershell
Get-Service | Where-Object { $_.Name -like '*Dell*' -or $_.Name -like '*SupportAssist*' -or $_.Name -like '*DDV*' }
```

All should be `Stopped`.

Then check registry:

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*', `
                 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue | `
    Where-Object { $_.Publisher -like '*Dell*' -and $_.DisplayName -notlike '*Update*' } | `
    Select-Object DisplayName | Format-Table -AutoSize
```

Only `Dell Update for Windows Universal` should remain.

---

## Phase 6 — Optional: Remove Dell Update Completely

If you **do not need BIOS updates** and want a fully clean machine:

```powershell
# Stop the Update Service
sc stop "Dell Update Service"  # (find actual service name first via Get-Service)

# Remove the x86 Dell Update folder
Remove-Item -Path "C:\Program Files (x86)\Dell" -Recurse -Force
```

---

## Troubleshooting

### Access denied on `sc stop` or `sc config`

**Cause:** Not running as Administrator.

**Fix:** Use `Start-Process sc -Verb RunAs -Wait` or open an elevated PowerShell prompt (Run as Administrator).

### A service restarts immediately after I stop it

**Cause:** The service is set to `Automatic` (StartType=2) instead of `Disabled`. Stop it first, then immediately run `sc config <name> start= disabled`.

### A Dell process appears in Task Manager even after cleanup

**Cause:** The process is likely started by a scheduled task or Windows service that wasn't disabled.

**Fix:** Check Task Scheduler for Dell tasks (`taskschd.msc`) and disable/remove them.

### After reboot, Dell software reappeared

**Cause:** Either the MSI uninstaller didn't fully remove dependent services, or Dell Update reinstalled components.

**Fix:** The DDV services are the most common culprit — they are separate from SupportAssist and survive the uninstall. Use Phase 2 to stop/disable them, then Phase 3 to remove registry entries, then Phase 4 to remove folders.

---

## Key Lessons

1. **SupportAssist MSI uninstaller does NOT remove DDV services.** Always check for DDV specifically.
2. **DDV services auto-start (StartType=2) from a separate `C:\Program Files\Dell\DellDataVault` folder.**
3. **Multiple registry uninstall stubs** survive in both `HKLM\Uninstall` and `HKLM\WOW6432Node\Uninstall` hives.
4. **Both x64 and x86 Dell folders exist** (`C:\Program Files\Dell` and `C:\Program Files (x86)\Dell`). The x86 folder holds the real Dell Update engine.
5. **The order should always be:** Identify → Stop → Disable → Delete registry → Remove files → Reboot → Verify.

---

## Quick Reference: Commands to Run

```
# 1. DISCOVER
Get-Service | Where-Object { $_.Name -like '*Dell*' -or $_.Name -like '*DDV*' }
tasklist | findstr /i /c:"dell" /c:"ddv" /c:"vault"

# 2. STOP + DISABLE (run each service name)
sc stop <ServiceName>
sc config <ServiceName> start= disabled

# 3. DELETE REGISTRY (use PSChildName from step 1.3)
reg delete "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{<GUID>}" /f

# 4. REMOVE FOLDERS
rmdir /s /q "C:\Program Files\Dell\DellDataVault"
rmdir /s /q "C:\Program Files\Dell\SupportAssistAgent"
rmdir /s /q "C:\Program Files\Dell\TechHub"

# 5. REBOOT AND VERIFY
tasklist | findstr /i /c:"dell" /c:"ddv"
# Should be empty
```
