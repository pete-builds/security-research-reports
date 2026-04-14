---
title: "Updating Computers with Microsoft's Unified Write Filter (UWF)"
date: 2026-04-01
summary: "Enterprise device hardening using Microsoft's Unified Write Filter for managed update deployments."
---

# Updating Computers with Microsoft's Unified Write Filter (UWF)

**Research Date:** April 1, 2026, 3:35 PM ET
**Requested by:** Pete
**Agent:** Research

---

## Current Status

Microsoft's recommended approach for updating UWF-protected devices is **UWF Servicing Mode**, a built-in mechanism that temporarily disables write filtering, installs updates, and re-enables protection automatically. This has been available since Windows 10 and carries forward into Windows 11. The feature requires the KB5015807 QFE (July 2022) on Windows 10 IoT images released before 2023. Remote management is also possible through the **UnifiedWriteFilter CSP** for MDM/Intune deployments.

---

## Overview: What UWF Is and Why Updates Are Tricky

Unified Write Filter (UWF) is an optional Windows feature that protects drives by intercepting and redirecting all write operations to a virtual overlay. On reboot, the overlay is cleared and the system returns to its original state. This is ideal for kiosks, thin clients, shared workstations, and IoT devices.

The core problem for updates: any changes made while UWF is active (including Windows Updates, antivirus signatures, and application patches) are stored only in the overlay and **discarded on the next reboot**. Standard patching workflows simply do not persist.

**Supported Editions** ([Microsoft Learn: UWF Overview](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/)):

| Edition | Supported |
|---------|-----------|
| Windows Home | No |
| Windows Pro | No |
| Windows Enterprise | Yes |
| Windows Education | Yes |
| Windows IoT Enterprise | Yes |

---

## Recommended Approach: UWF Servicing Mode

UWF Servicing Mode is Microsoft's official solution. It automates the full cycle: disable write protection, install updates, re-enable protection, and reboot. ([Microsoft Learn: Apply Windows Updates to UWF-Protected Devices](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-apply-windows-updates))

### How It Works

1. An administrator enables servicing mode via command line or script.
2. The device reboots.
3. On restart, the system automatically signs into a built-in local account called **UWF-Servicing**.
4. UWF is temporarily disabled, and the overlay is cleared.
5. Windows Update scans for and installs critical updates, security updates, driver updates, and antimalware signature updates.
6. If a restart is required by updates, the system re-enters servicing mode and continues installing.
7. When all updates are complete, UWF servicing is disabled, UWF protection is re-enabled, and all file/registry exclusions are restored.
8. The system reboots into normal protected operation.

No user interaction is required during the process. A screen saver (`UwfServicingScr.scr`) displays while servicing is active. ([Microsoft Learn: Apply Windows Updates](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-apply-windows-updates))

### Supported Update Types

- Critical updates
- Security updates
- Driver updates
- Antimalware signature files

### Prerequisites

- **KB5015807** must be installed on Windows 10 IoT images released before 2023. Images from 2023 onward include this fix. ([Dell: Expert Insights into UWF Servicing Mode](https://www.dell.com/support/kbdoc/en-us/000222266/expert-insights-into-unified-write-filter-servicing-mode))
- The device must be able to reach either Microsoft Windows Update servers (internet) or an internal WSUS server.
- Do not create a local user account named "UWF-Servicing" on the device, as this conflicts with the built-in servicing account.

### Triggering Servicing Mode

**Command line (recommended):**

```cmd
uwfmgr.exe servicing enable
uwfmgr.exe filter restart
```

Or simply:

```cmd
uwfmgr.exe servicing enable
shutdown /r /t 0
```

**Master servicing script:** The system uses `UwfServicingMasterScript.cmd` internally. This script can be customized to also service third-party applications or call custom OEM servicing scripts. ([Microsoft Learn: Apply Windows Updates](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-apply-windows-updates))

**Manual batch file (Dell thin clients):** On Dell/Wyse thin clients running Windows 10 IoT, you can run: `C:\Windows\System32\oem\SystemServicing.bat` ([Dell: Expert Insights](https://www.dell.com/support/kbdoc/en-us/000222266/expert-insights-into-unified-write-filter-servicing-mode))

**Scheduled task:** A pre-built but disabled scheduled task (`WindowsUWFServicing`) exists on compatible images. Enable it with:

```powershell
Enable-ScheduledTask -TaskName "WindowsUWFServicing"
```

You can then configure when and how frequently it runs. ([Dell: Expert Insights](https://www.dell.com/support/kbdoc/en-us/000222266/expert-insights-into-unified-write-filter-servicing-mode))

**Standalone update command (without full servicing cycle):**

```cmd
uwfmgr.exe servicing update-windows
```

This calls Windows Update directly but Microsoft recommends using `servicing enable` for the full automated cycle whenever possible. ([Microsoft Learn: uwfmgr.exe](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwfmgrexe))

---

## Alternative Approaches

### Manual Disable/Update/Re-enable

For one-off maintenance or when servicing mode is not suitable:

```cmd
REM Disable UWF protection (takes effect after reboot)
uwfmgr.exe filter disable

REM Reboot
shutdown /r /t 0

REM ... install updates manually or via WSUS/SCCM ...

REM Re-enable UWF protection (takes effect after reboot)
uwfmgr.exe filter enable

REM Reboot
shutdown /r /t 0
```

This approach requires two reboots at minimum and leaves the system unprotected during the maintenance window. It is more error-prone than servicing mode because forgetting to re-enable UWF leaves the device permanently unprotected. ([Windows OS Hub: Using UWF on Windows 10](https://woshub.com/using-unified-write-filter-uwf-windows-10/))

### File and Registry Exclusions

For narrow, ongoing updates (like antivirus signature files), you can add specific paths and registry keys to the UWF exclusion list. Writes to excluded paths persist across reboots without disabling UWF:

```cmd
uwfmgr.exe file add-exclusion C:\ProgramData\Microsoft\Windows Defender\Definition Updates
uwfmgr.exe registry add-exclusion "HKLM\SOFTWARE\Microsoft\Windows Defender"
```

**Warning:** Overusing exclusions undermines UWF's protection. Improper exclusions (especially to boot-related files) can cause boot loops or Stop errors. Microsoft specifically warns against excluding `%windir%\bootstat.dat`. ([Microsoft Learn: UWF Overview](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/))

### Commit Individual Files

For small, targeted changes, you can commit specific files from the overlay to the physical volume without disabling UWF:

```cmd
uwfmgr.exe file commit C:\path\to\specific\file.dat
```

This is useful for one-off configuration changes but impractical for Windows Update, which touches hundreds of files. ([Microsoft Learn: uwfmgr.exe](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwfmgrexe))

---

## Integration with Management Tools

### WSUS (Windows Server Update Services)

UWF Servicing Mode works natively with WSUS. For devices on restricted networks without internet access, configure the WSUS server via Group Policy:

**Group Policy path:** Computer Configuration > Administrative Templates > Windows Components > Windows Update > Specify intranet Microsoft update service location

**PowerShell/registry approach:**

```powershell
$Path = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
Set-ItemProperty -Path $Path -Name "WUServer" -Value "https://wsusserver.yourdomain.com"
Set-ItemProperty -Path $Path -Name "WUStatusServer" -Value "https://wsusserver.yourdomain.com"
Set-ItemProperty -Path $Path -Name "TargetGroup" -Value "UWF-Devices"
Set-ItemProperty -Path $Path -Name "TargetGroupEnabled" -Value 1
```

([Dell: Expert Insights](https://www.dell.com/support/kbdoc/en-us/000222266/expert-insights-into-unified-write-filter-servicing-mode))

### SCCM / ConfigMgr

Configuration Manager is "UWF aware" and simplifies patching on UWF-protected devices. Key considerations:

- **Add the ccmcache folder as a UWF exclusion** so the ConfigMgr client cache persists across reboots. Without this, downloaded update content is lost on each reboot.
- ConfigMgr client files should also be excluded to maintain the management agent's state.
- Plan maintenance windows in ConfigMgr to coordinate with UWF servicing mode.

([Recast Software: UWF Overview](https://www.recastsoftware.com/resources/unified-write-filter-basic-overview/))

### Intune / MDM (UnifiedWriteFilter CSP)

Microsoft provides the **UnifiedWriteFilter CSP** for remote management via MDM solutions including Intune. ([Microsoft Learn: UnifiedWriteFilter CSP](https://learn.microsoft.com/en-us/windows/client-management/mdm/unifiedwritefilter-csp))

Key OMA-URI paths for servicing:

| OMA-URI | Purpose | Operations |
|---------|---------|------------|
| `./Vendor/MSFT/UnifiedWriteFilter/CurrentSession/ServicingEnabled` | Check if servicing is active now | Get |
| `./Vendor/MSFT/UnifiedWriteFilter/NextSession/ServicingEnabled` | Enable/disable servicing for next session | Get, Replace |
| `./Vendor/MSFT/UnifiedWriteFilter/NextSession/FilterEnabled` | Enable/disable UWF for next session | Get, Replace |
| `./Vendor/MSFT/UnifiedWriteFilter/RestartSystem` | Safe restart (even if overlay is full) | Get, Execute |

In Intune, configure these using a **Custom Configuration Profile** with OMA-URI settings. Set `NextSession/ServicingEnabled` to `true`, then trigger a restart to enter servicing mode remotely.

---

## PowerShell/CLI Reference

### uwfmgr.exe Quick Reference

| Command | Description |
|---------|-------------|
| `uwfmgr.exe get-config` | Display full UWF configuration (current and next session) |
| `uwfmgr.exe filter enable` | Enable UWF protection (next reboot) |
| `uwfmgr.exe filter disable` | Disable UWF protection (next reboot) |
| `uwfmgr.exe filter restart` | Restart the system (safe, even with full overlay) |
| `uwfmgr.exe volume protect C:` | Add C: to protected volumes |
| `uwfmgr.exe volume unprotect C:` | Remove C: from protected volumes |
| `uwfmgr.exe servicing enable` | Enter servicing mode on next reboot |
| `uwfmgr.exe servicing disable` | Cancel pending servicing mode |
| `uwfmgr.exe servicing update-windows` | Directly trigger Windows Update |
| `uwfmgr.exe servicing get-config` | Show servicing mode status |
| `uwfmgr.exe overlay get-config` | Show overlay type, size, thresholds |
| `uwfmgr.exe overlay get-consumption` | Show current overlay usage |
| `uwfmgr.exe overlay get-availablespace` | Show remaining overlay space |
| `uwfmgr.exe file add-exclusion <path>` | Exclude a file/folder from filtering |
| `uwfmgr.exe file commit <path>` | Write a file from overlay to physical disk |
| `uwfmgr.exe registry add-exclusion <key>` | Exclude a registry key from filtering |

([Microsoft Learn: uwfmgr.exe](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwfmgrexe))

### WMI/PowerShell for Servicing Mode

```powershell
$COMPUTER = "localhost"
$NAMESPACE = "root\standardcimv2\embedded"
$CommonParams = @{"namespace"=$NAMESPACE; "computer"=$COMPUTER}

# Enable servicing mode for next session
$nextSession = Get-WmiObject -class UWF_Servicing @CommonParams |
    Where-Object { $_.CurrentSession -eq $false }
if ($nextSession) {
    $nextSession.Enable() | Out-Null
    Write-Host "Servicing mode enabled for next restart."
}

# Disable servicing mode
$nextSession = Get-WmiObject -class UWF_Servicing @CommonParams |
    Where-Object { $_.CurrentSession -eq $false }
if ($nextSession) {
    $nextSession.Disable() | Out-Null
    Write-Host "Servicing mode disabled."
}
```

([Microsoft Learn: UWF_Servicing](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-servicing))

### Install UWF Feature via PowerShell

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName "Client-UnifiedWriteFilter" -All
```

---

## Best Practices and Gotchas

### Best Practices

1. **Use servicing mode, not manual disable/enable.** Servicing mode handles the full cycle automatically and reduces the risk of leaving devices unprotected.
2. **Schedule servicing during maintenance windows.** Coordinate with your organization's patching schedule. The built-in `WindowsUWFServicing` scheduled task can automate this.
3. **Monitor overlay consumption.** Set warning and critical thresholds so you know when the overlay is filling up. A full overlay can freeze or crash the device.
4. **Keep exclusions minimal.** Every exclusion is a gap in UWF's protection. Only exclude what is strictly necessary (ConfigMgr cache, AV signatures, specific logs).
5. **Test thoroughly in a lab first.** Especially test exclusion changes and servicing scripts before rolling out to production kiosks or thin clients.
6. **Disable Fast Startup.** UWF does not clear the overlay on shutdown when Fast Startup is enabled. Navigate to Control Panel > Power Options > System Settings and uncheck "Turn on fast startup." ([Microsoft Learn: UWF Overview](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/))
7. **Use WSUS targeting for UWF devices.** Create a dedicated WSUS computer group for UWF-protected devices so you can control which updates are approved for them.

### Gotchas

1. **The UWF-Servicing account conflict.** Do not create a local user account named "UWF-Servicing" on any device. The built-in servicing process requires this account name. ([Microsoft Learn: Apply Windows Updates](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-apply-windows-updates))
2. **KB5015807 is mandatory for older Win10 IoT images.** Without this update, servicing mode will not function on pre-2023 images. ([Dell: Expert Insights](https://www.dell.com/support/kbdoc/en-us/000222266/expert-insights-into-unified-write-filter-servicing-mode))
3. **Network connectivity is non-negotiable.** Servicing mode requires access to Windows Update servers or a WSUS server. No connection means no updates, and servicing will fail silently, re-enabling UWF without applying anything.
4. **Failed updates still re-enable UWF.** If Windows updates cannot be installed or return an error, servicing is automatically disabled and UWF protection is re-enabled. Check Windows Update logs to diagnose failures. ([Microsoft Learn: Apply Windows Updates](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-apply-windows-updates))
5. **Do not exclude bootstat.dat.** Adding `%windir%\bootstat.dat` to the exclusion list causes Stop error 0x7E (SYSTEM_THREAD_EXCEPTION_NOT_HANDLED). ([Microsoft Learn: UWF Overview](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/))
6. **Windows Defender exclusions can trigger boot loops.** Adding broad Defender directory exclusions requires careful testing. ([Windows OS Hub](https://woshub.com/using-unified-write-filter-uwf-windows-10/))
7. **Date/time changes do not persist.** On UWF-protected devices, setting the date/time to a past value will revert after reboot. You must disable UWF entirely to make permanent time changes. ([Microsoft Learn: UWF Overview](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/))
8. **SCCM ccmcache must be excluded.** Without this exclusion, downloaded update packages are lost on reboot, causing repeated download cycles. ([Recast Software](https://www.recastsoftware.com/resources/unified-write-filter-basic-overview/))
9. **License terms are auto-accepted.** During UWF servicing in Windows 10 Enterprise, Windows Update automatically accepts all Microsoft Software License Terms. ([Microsoft Learn: Apply Windows Updates](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-apply-windows-updates))

### Windows 10 vs. Windows 11 Differences

UWF is functionally the same across Windows 10 and Windows 11 Enterprise/Education editions. The same `uwfmgr.exe` commands, WMI classes, and CSP paths apply to both. The main consideration is that Windows 11 requires Enterprise or Education edition (same as Windows 10). The `filter reset-settings` command is no longer supported starting in Windows 10. ([Microsoft Learn: uwfmgr.exe](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwfmgrexe))

---

## Sources

- [Microsoft Learn: Unified Write Filter (UWF) Overview](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/)
- [Microsoft Learn: Apply Windows Updates to UWF-Protected Devices](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-apply-windows-updates)
- [Microsoft Learn: UWF_Servicing WMI Class](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwf-servicing)
- [Microsoft Learn: uwfmgr.exe Command Reference](https://learn.microsoft.com/en-us/windows/configuration/unified-write-filter/uwfmgrexe)
- [Microsoft Learn: UnifiedWriteFilter CSP (MDM/Intune)](https://learn.microsoft.com/en-us/windows/client-management/mdm/unifiedwritefilter-csp)
- [Dell: Expert Insights into UWF Servicing Mode](https://www.dell.com/support/kbdoc/en-us/000222266/expert-insights-into-unified-write-filter-servicing-mode)
- [Recast Software: UWF Basic Overview (SCCM Integration)](https://www.recastsoftware.com/resources/unified-write-filter-basic-overview/)
- [Windows OS Hub: Using UWF on Windows 10](https://woshub.com/using-unified-write-filter-uwf-windows-10/)
