---
title: "Signed Driver Used to Kill CrowdStrike EDR"
date: 2026-04-14
updated: 2026-04-14
summary: "IOCTL 0x22E010 is a vendor-defined buffered I/O control code used by the Rentdrv2/PoisonX kernel driver to terminate arbitrary processes, including PPL-protected EDR services. It is associated with CVE-2023-44976 and has been weaponized in BYOVD attacks by the Agonizing Serpens (Agrius) APT group since October 2023."
---

# Signed Driver Used to Kill CrowdStrike EDR
### Reverse Engineering IOCTL 0x22E010: The EDR-Killing Control Code

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Current Status](#current-status)
3. [IOCTL Bit Field Breakdown](#ioctl-bit-field-breakdown)
4. [The Driver: Rentdrv2 / PoisonX](#the-driver-rentdrv2--poisonx)
5. [Attack Pattern: BYOVD](#attack-pattern-byovd-bring-your-own-vulnerable-driver)
6. [Reverse Engineering Methodology](#reverse-engineering-methodology)
7. [Security Analysis](#security-analysis)
   - [CVE-2023-44976](#cve-2023-44976)
   - [Threat Landscape](#threat-landscape)
   - [Detection Opportunities](#detection-opportunities)
   - [Why the CVSS 3.2 "Low" Score is Misleading](#why-the-cvss-32-low-score-is-misleading)
8. [IT Operations Guide](#it-operations-guide)
   - [Immediate Action Checklist](#immediate-action-checklist)
   - [Check Your Protection Status](#check-your-protection-status)
   - [Known Driver Hashes for Blocklisting](#known-driver-hashes-for-blocklisting)
   - [Event Log Detection](#event-log-detection)
   - [Detection Rules](#detection-rules)
   - [Forensic Triage](#forensic-triage-has-this-already-happened)
   - [Real-World Threat Actor Usage](#real-world-threat-actor-usage)
9. [Confidence Assessment](#confidence-assessment)
10. [Sources](#sources)

---

## Executive Summary

IOCTL 0x22E010 is a process termination control code exposed by the Rentdrv2 driver (Hangzhou Shunwang Technology). A user-mode application sends the IOCTL to the driver, which then executes process termination from kernel mode, bypassing user-mode access restrictions such as PPL (Protected Process Light) enforcement that would block ordinary process open/terminate attempts. This allows it to kill processes including CrowdStrike Falcon and other EDR products. The driver carries a valid Microsoft signature and had zero antivirus detections at time of discovery.

A vendor patch exists (versions after 2024-12-24) that reduces exposure for legitimate deployments, but it does not by itself neutralize BYOVD (Bring Your Own Vulnerable Driver) use of older signed copies. In BYOVD attacks, adversaries drop their own copy of the old, pre-patch signed binary onto the target system. Because the Microsoft signature remains valid, Windows loads it without complaint. Organizations still need blocklisting or App Control enforcement to prevent those binaries from loading. This technique has been weaponized in the wild since October 2023 by the Iranian APT group [Agonizing Serpens (Agrius)](https://unit42.paloaltonetworks.com/agonizing-serpens-targets-israeli-tech-higher-ed-sectors/) against Israeli targets.

**Why this is in the news now (April 2026):** The vulnerability has been exploited since October 2023, but a [full reverse engineering walkthrough](https://medium.com/@jehadbudagga/reverse-engineering-a-0day-used-against-crowdstrike-edr-a5ea1fbe3fd4) and working proof-of-concept ([PoisonKiller](https://github.com/Nekr0w/poisonkiller)) were publicly released this week. Ten PoisonX driver variants were [cataloged on LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) on April 10. The barrier to replicate this attack has dropped significantly.

**What you can do now:** Confirm [HVCI](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/microsoft-recommended-driver-block-rules) is enabled on your endpoints, verify the Microsoft Vulnerable Driver Blocklist is current, and enable the Defender ASR rule "Block abuse of exploited vulnerable signed drivers" (GUID: `56a863a9-875e-4185-98a7-b882c64b5ce5`). Note: the ASR rule prevents writing a vulnerable driver to disk but does not block loading one already present. Use it alongside HVCI and the blocklist. PowerShell commands and detection rules are in the [IT Operations Guide](#it-operations-guide) below.

At the ecosystem level, the strongest mitigation would be certificate revocation or inclusion in Microsoft's [Vulnerable Driver Blocklist](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/microsoft-recommended-driver-block-rules). At the organization level, HVCI (Hypervisor-protected Code Integrity), App Control for Business, and an up-to-date driver blocklist are the most practical controls. Detection should focus on driver loads, the device path `\\.\{F8284233-48F4-4680-ADDD-F8284233}`, and unexpected EDR telemetry loss.

---

## Current Status

**Confirmed facts:**
- **IOCTL 0x22E010** is a process-termination control code exposed by the Rentdrv2/PoisonX Windows kernel driver
- Assigned **CVE-2023-44976** (CWE-782: Exposed IOCTL with Insufficient Access Control), CVSS 3.2 Low per MITRE-provided vector
- Exploited in the wild since **October 2023** by the Iranian-backed APT group Agonizing Serpens (Agrius) in attacks against Israeli higher education and tech sectors ([Palo Alto Unit 42](https://unit42.paloaltonetworks.com/agonizing-serpens-targets-israeli-tech-higher-ed-sectors/))
- Known vulnerable Rentdrv2/PoisonX variants have been observed with valid Microsoft signatures. Signature metadata varies by sample.
- Vendor patch available: Rentdrv2 versions after 2024-12-24 address the vulnerability ([NVD](https://nvd.nist.gov/vuln/detail/CVE-2023-44976))

**Time-sensitive observations (as of April 2026):**
- Multiple public proof-of-concept tools now available: [PoisonKiller](https://github.com/Nekr0w/poisonkiller), [BadRentdrv2](https://github.com/keowu/BadRentdrv2), [PoisonX-Killer](https://github.com/BlackSnufkin/BYOVD/blob/main/PoisonX-Killer/src/main.rs)
- One variant was reported as **0/71 on VirusTotal** at the time the PoC author documented the sample; detection rates are inherently unstable and may have changed ([GitHub/Nekr0w](https://github.com/Nekr0w/poisonkiller))
- Ten PoisonX variants [cataloged on LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) as of April 10, 2026

---

## IOCTL Bit Field Breakdown

The Windows [`CTL_CODE`](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/defining-i-o-control-codes) macro encodes four fields into a 32-bit IOCTL value:

```
CTL_CODE(DeviceType, Function, Method, Access)
```

### Decoding 0x0022E010

```
Hex:    0x0022E010
Binary: 0000 0000 0010 0010 1110 0000 0001 0000

Bits 31-16 (Device Type):  0x0022 = 34 = FILE_DEVICE_UNKNOWN
Bits 15-14 (Access):       0x3    = 3  = FILE_READ_ACCESS | FILE_WRITE_ACCESS
Bits 13-2  (Function Code): 0x804 = 2052 (vendor-defined, >= 0x800)
Bits 1-0   (Method):       0x0    = 0  = METHOD_BUFFERED
```

### Interpretation

| Field | Value | Meaning |
|-------|-------|---------|
| Device Type | `FILE_DEVICE_UNKNOWN` (0x0022) | Generic/unspecified device type, common for third-party drivers that don't fit a standard category |
| Required Access | `FILE_READ_ACCESS \| FILE_WRITE_ACCESS` (0x3) | Caller must have both read and write access to the device object |
| Function Code | 0x804 (2052) | Vendor-defined function. Microsoft reserves 0x000-0x7FF for standard functions; 0x800+ is for third-party use |
| Transfer Method | `METHOD_BUFFERED` (0x0) | Kernel copies input/output buffers via system buffer (Irp->AssociatedIrp.SystemBuffer), the safest transfer method |

### Reconstruction verification

```c
CTL_CODE(FILE_DEVICE_UNKNOWN, 0x804, METHOD_BUFFERED, FILE_READ_DATA | FILE_WRITE_DATA)
= (0x0022 << 16) | (0x3 << 14) | (0x804 << 2) | 0x0
= 0x00220000 | 0x0000C000 | 0x00002010 | 0x0
= 0x0022E010  // Confirmed
```

**WinDbg verification command:**
```
!ioctldecode 0x22E010
```

---

## The Driver: Rentdrv2 / PoisonX

### Identity

IOCTL 0x22E010 belongs to the **Rentdrv2** driver, developed by Hangzhou Shunwang Technology. The driver has also been referred to as **PoisonX.sys** in PoC tooling. There are reportedly 15+ variants sharing identical code ([Medium/@jehadbudagga](https://medium.com/@jehadbudagga/reverse-engineering-a-0day-used-against-crowdstrike-edr-a5ea1fbe3fd4)).

| Property | Value |
|----------|-------|
| Original Name | Rentdrv2.sys |
| PoC Name | PoisonX.sys |
| Vendor | Hangzhou Shunwang Technology |
| Signature | Microsoft Windows Hardware Compatibility Publisher |
| Device Path | `\\.\{F8284233-48F4-4680-ADDD-F8284233}` |
| VT Detection | 0/71 (at time PoC author documented the sample; may have changed) |

### What the driver does

Rentdrv2 is a legitimate internet cafe management driver used in China. Its intended purpose is to manage user sessions and system resources in rental computer environments. However, its IOCTL dispatch handler exposes a process termination function that executes from kernel mode, bypassing user-mode access restrictions such as PPL enforcement that would block ordinary user-mode process open/terminate attempts.

### The kill mechanism

When IOCTL 0x22E010 is received, the driver:

1. Reads the input buffer, which contains a PID as a decimal ASCII string
2. Calls `ZwOpenProcess` to obtain a kernel handle to the target process
3. Calls `ZwTerminateProcess` to kill the process

This is significant because `ZwOpenProcess` called from kernel mode bypasses Process Protection Light (PPL) restrictions. In user mode, `OpenProcess` against a PPL-protected process returns ACCESS_DENIED, but the kernel-mode `Zw*` functions operate with full privilege ([Medium/@jehadbudagga](https://medium.com/@jehadbudagga/reverse-engineering-a-0day-used-against-crowdstrike-edr-a5ea1fbe3fd4)).

### Input/Output format

```
Input:  PID as decimal ASCII string (e.g., "1234"), sent as bytes
Output: 16-byte buffer; returns "ok" on success
Method: METHOD_BUFFERED (via Irp->AssociatedIrp.SystemBuffer)
```

---

## Attack Pattern: BYOVD (Bring Your Own Vulnerable Driver)

### How it works

1. **Drop the driver**: Attacker places Rentdrv2.sys/PoisonX.sys on disk
2. **Load via service**: Create a kernel service (`sc create ... type=kernel binPath=...`) and start it
3. **Open device**: Call `CreateFileW` with device path `\\.\{F8284233-48F4-4680-ADDD-F8284233}`
4. **Send kill IOCTL**: Call `DeviceIoControl` with control code 0x22E010, passing the target PID as input
5. **EDR process terminated**: The driver executes termination from kernel mode, bypassing PPL restrictions that would block user-mode attempts against CrowdStrike Falcon, Defender, and similar protected processes

The driver loads without triggering code-integrity checks because it carries a valid Microsoft signature ([GitHub/Nekr0w](https://github.com/Nekr0w/poisonkiller)).

### Real-world usage

**Agonizing Serpens (Agrius)** - an Iranian-backed APT group - weaponized Rentdrv2 in attacks against Israeli higher education and technology sector organizations starting in October 2023. The group used a custom loader called `drvIX.exe` to communicate with the driver. Palo Alto's Unit 42 documented this as an upgrade in the group's capabilities, noting they had not used BYOVD techniques in previous campaigns ([Palo Alto Unit 42](https://unit42.paloaltonetworks.com/agonizing-serpens-targets-israeli-tech-higher-ed-sectors/)).

---

## Reverse Engineering Methodology

### Tools for IOCTL analysis

| Tool | Purpose | Notes |
|------|---------|-------|
| **IDA Pro** | Static analysis/decompilation of the .sys driver | Primary tool used in the original RE analysis |
| **Ghidra** | Free alternative to IDA for driver decompilation | NSA's reverse engineering framework |
| **WinDbg** | Kernel debugging, [`!ioctldecode`](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-ioctldecode) command | Built-in IOCTL decoder: `!ioctldecode 0x22E010` |
| **WinObj** (Sysinternals) | Discover device symbolic links at runtime | Used to find `\\.\{F8284233-48F4-4680-ADDD-F8284233}` |
| **OSR IOCTL Decoder** | Online tool to decode CTL_CODE fields | Quick bit field breakdown |
| **OSR Driver Loader** | Load/unload kernel drivers for testing | Used instead of `sc create` for lab environments |
| **[IoctlHunter](https://z4ksec.github.io/posts/ioctlhunter-release-v0.2/)** | Dynamic IOCTL monitoring via DLL injection | Captures IOCTLs in a controlled lab environment |
| **VirusTotal** | Check driver signature and detection rate | Confirmed valid Microsoft signature |

### RE approach for this driver

The original reverse engineer (Jehad Abudagga) documented the following approach ([Medium/@jehadbudagga](https://medium.com/@jehadbudagga/reverse-engineering-a-0day-used-against-crowdstrike-edr-a5ea1fbe3fd4)):

1. **Skip the DriverEntry**: The `DriverEntry` function was heavily obfuscated/mangled. Instead of fighting the obfuscation, the analyst went directly to the **IRP dispatch handler** (the function registered for `IRP_MJ_DEVICE_CONTROL`)
2. **Fix types and names**: IDA's initial decompilation output was difficult to read. Manual type annotation and variable renaming were required to produce clean pseudocode
3. **Identify the IOCTL dispatch table**: The dispatch handler compared incoming IOCTL codes and routed to different functions. Two IOCTLs were found; 0x22E010 was the one leading to the process kill function
4. **Trace the kill chain**: Following the 0x22E010 handler revealed the `ZwOpenProcess` to `ZwTerminateProcess` call sequence

### General IOCTL RE methodology

For reverse engineering any unknown IOCTL:

1. **Decode the bits**: Use `CTL_CODE` macro decomposition (as shown above) to understand device type, access requirements, function code, and transfer method
2. **Identify the driver**: Search for the IOCTL value in known databases (ioctls.net, the DDK headers, public malware/PoC repos)
3. **Load in disassembler**: Open the .sys file in IDA/Ghidra, locate `DriverEntry`, find the `IRP_MJ_DEVICE_CONTROL` dispatch function
4. **Find the switch/comparison**: The dispatch handler will compare `IoStackLocation->Parameters.DeviceIoControl.IoControlCode` against known values
5. **Trace the handler**: Follow the code path for your target IOCTL to understand what it does
6. **Dynamic analysis**: Use WinDbg kernel debugging to set breakpoints on `NtDeviceIoControlFile` and observe live IOCTL traffic

---

## Security Analysis

### CVE-2023-44976

| Field | Value |
|-------|-------|
| CVE ID | [CVE-2023-44976](https://nvd.nist.gov/vuln/detail/CVE-2023-44976) |
| CWE | CWE-782: Exposed IOCTL with Insufficient Access Control |
| CVSS 3.1 | 3.2 (Low) per MITRE-provided vector |
| Vector | AV:L/AC:L/PR:H/UI:N/S:C/C:N/I:N/A:L |
| Affected | Hangzhou Shunwang Rentdrv2 before 2024-12-24 |
| Exploited in Wild | October 2023 |
| NVD Published | August 1, 2025 |
| Fix | Update to Rentdrv2 versions released 2024-12-24 or later |

Source: [NVD CVE-2023-44976](https://nvd.nist.gov/vuln/detail/CVE-2023-44976), [Feedly CVE feed](https://feedly.com/cve/CVE-2023-44976)

**Note on CVSS score**: The 3.2 Low rating (from the MITRE-provided vector in the CVE record; NVD's own assessment fields currently show N/A) is arguably understated. While the attack vector is local and requires high privileges (admin to load a driver), the scope is changed (S:C) because it affects processes outside the vulnerable component's security context. The real-world impact of killing EDR processes to enable further attacks is substantially higher than the base score suggests.

### Threat landscape

- The vulnerability sat in **reserved CVE status for over a year** before publication (wild exploitation October 2023, NVD published August 2025) ([Feedly](https://feedly.com/cve/CVE-2023-44976))
- Multiple public PoC repositories exist on GitHub, lowering the barrier for copycat attacks
- The BYOVD technique using this driver is potentially effective against EDR products whose protection model depends heavily on keeping critical services protected from user-mode termination, not just CrowdStrike. Individual products may layer additional self-defense logic, kernel callbacks, or recovery mechanisms. ([GitHub/Muz1K1zuM](https://github.com/Muz1K1zuM/PoisonKiller_bof))

### Detection opportunities

1. **Driver load monitoring**: Alert on PoisonX.sys, Rentdrv2.sys, or service creation for unfamiliar kernel drivers
2. **Device path monitoring**: Watch for `CreateFile` calls targeting `\\.\{F8284233-48F4-4680-ADDD-F8284233}`
3. **IOCTL monitoring**: Detect `DeviceIoControl` calls with code 0x22E010
4. **Driver hash blocklisting**: Add known Rentdrv2/PoisonX driver hashes to WDAC or HVCI blocklists
5. **EDR telemetry loss**: Prioritize agent heartbeat-loss detection. If an EDR agent goes silent unexpectedly, treat that as a high-priority indicator rather than trying to infer kernel-initiated termination directly

### Why the CVSS 3.2 "Low" score is misleading

The base score rates this as low-severity because the attack vector is local and requires admin privileges. In practice, this drastically understates the risk. Attackers who already have admin access use this driver as a force multiplier: they kill EDR first, then deploy ransomware, exfiltrate data, or persist undetected. The CVSS score measures the driver in isolation. The real-world impact is the chain: EDR dies, then the actual attack begins with no visibility. DragonForce ransomware uses Rentdrv2.sys as one of its two BYOVD backends (config field `use_sys=2`), confirming this is an active component in ransomware kill chains, not a theoretical risk ([S2W Medium](https://medium.com/s2wblog/detailed-analysis-of-dragonforce-ransomware-25d1a91a4509), [Acronis TRU](https://www.acronis.com/en/tru/posts/the-dragonforce-cartel-scattered-spider-at-the-gate/)).

---

## IT Operations Guide

This section is written for IT staff who need to assess exposure, detect this attack, and harden their environment.

### Immediate action checklist

1. **Verify HVCI is enabled** on all Windows endpoints (see commands below)
2. **Verify the Microsoft Vulnerable Driver Blocklist is active** and current
3. **Enable the ASR rule** "Block abuse of exploited vulnerable signed drivers" (GUID: `56a863a9-875e-4185-98a7-b882c64b5ce5`) ([Microsoft Learn](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-reference)). Important: this rule helps prevent a vulnerable driver from being written to disk, but it does not stop loading of a driver that is already present on the system.
4. **Deploy Sysmon** if not already running, with driver load logging (Event ID 6)
5. **Search historical logs** for indicators of past compromise (see Forensic Triage below)
6. **Add driver hashes** to your blocklist if using a custom WDAC policy

### Check your protection status

**Check HVCI / Virtualization-Based Security:**
```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
```
Look for `VirtualizationBasedSecurityStatus`: 2 = Running, 1 = Enabled but not running, 0 = Not enabled. ([Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/microsoft-recommended-driver-block-rules), [HotCakeX/Harden-Windows-Security](https://github.com/HotCakeX/Harden-Windows-Security/wiki/WDAC-Notes))

**Check WDAC / App Control policy status:**
```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard | Select-Object -Property *codeintegrity* | Format-List
```
Values: `2` = Enforced, `1` = Audit mode, `0` = Disabled. Both `UserModeCodeIntegrityPolicyEnforcementStatus` and `CodeIntegrityPolicyEnforcementStatus` (kernel mode) should be checked. ([HotCakeX/Harden-Windows-Security](https://github.com/HotCakeX/Harden-Windows-Security/wiki/WDAC-Notes))

**Refresh App Control policies after deploying blocklist updates:**
```powershell
CITool --refresh
```
([Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/microsoft-recommended-driver-block-rules))

**Note:** The Microsoft Vulnerable Driver Blocklist is enabled by default on Windows 11 (2022 update and later) and is enforced when HVCI, Smart App Control, or S mode is active. If you are running Windows 10 or have not enabled HVCI, the blocklist may not be enforced. ([Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/microsoft-recommended-driver-block-rules))

### Known driver hashes for blocklisting

**Rentdrv2.sys:**

| Variant | SHA256 | Source |
|---------|--------|--------|
| x64 | `9165d4f3036919a96b86d24b64d75d692802c7513f2b3054b20be40c212240a5` | [LOLDrivers](https://www.loldrivers.io/drivers/afb8bb46-1d13-407d-9866-1daa7c82ca63/), [BadRentdrv2 GitHub](https://github.com/keowu/BadRentdrv2) |
| x32 | `1aed62a63b4802e599bbd33162319129501d603cceeb5e1eb22fd4733b3018a3` | [BadRentdrv2 GitHub](https://github.com/keowu/BadRentdrv2) |

**PoisonX.sys variants** (all share Authentihash `1ce001eda18c7a0636c4053848a1ae69812704795ab057331903d7002d688027`):

| Variant | SHA256 | Source |
|---------|--------|--------|
| PoisonX.sys | `a5035cbd6c31616288aa66d98e5a25441ee38651fb5f330676319f921bb816a4` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX10.sys | `d58df93524ead1a1f939438ef63b5a9e42aacec7463ed29878293382996640ce` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX11.sys | `cc6b4e34a4c49ce2770615aa38b00ce479e236d34307f950cf5d8ad46b458627` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX12.sys | `c070b4a5614cb234de0faaef4e131b0cfe127682dc4cc89856e1ebfe775f4725` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX13.sys | `50870b82104a309c107ad6ed50c023324fd73b6908a52f6070f47b8d247323c9` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX14.sys | `6452ca681dd818a36ec538fe2d83a795f98e14970855964520294726dedd742b` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX15.sys | `846b2f282925d7ddccd41f46f0048bc1f424fe44ccb110ed45ece8637fbece5d` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX16.sys | `103b1bbe9d257e3ae6b83befdc53cad0e5432ade535b1f85dd6fddd4356d4898` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX17.sys | `6f24ed64cba4ed0901592770a3aead821317a85efa886bd51f1afc6cb1166990` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| PoisonX18.sys | `1d9ae1467e604469798d272755afcf845b5efcd588863ad9e2aa3e3cf112985a` | [LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/) |
| Unnamed variant | `6fbaad2f00afaa94723fa7d5bd46e7ea4babb7ce478a8e7229ce7bd4b85e0f51` | [Medium/@jehadbudagga](https://medium.com/@jehadbudagga/reverse-engineering-a-0day-used-against-crowdstrike-edr-a5ea1fbe3fd4) |

All PoisonX variants signed by "Microsoft Windows Hardware Compatibility Publisher" (cert serial `330000006e1229856f0ade6cfc00000000006e`, issued by Microsoft Windows Third Party Component CA 2014). ([LOLDrivers](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/))

### Event log detection

| Event ID | Log Source | What it catches | Notes |
|----------|-----------|----------------|-------|
| **[Sysmon 6](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon#event-id-6-driver-loaded)** | Sysmon | Driver loaded | Primary detection. Logs driver path, hashes, and signature. Requires Sysmon deployed. |
| **System 7045** | Service Control Manager | New service installed | Catches `sc create ... type=kernel`. Note: the [PoisonKiller_bof](https://github.com/Muz1K1zuM/PoisonKiller_bof) variant uses NtLoadDriver directly to bypass this event. |
| **[Security 4697](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4697)** | Security log | Service installed | Similar to 7045 but in Security log. Also bypassed by NtLoadDriver. |
| **[Sysmon 13](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon#event-id-13-registryevent-value-set)** | Sysmon | Registry value set | Catches `HKLM\SYSTEM\CurrentControlSet\Services\<name>` creation for driver registration. |
| **CodeIntegrity 3099** | CodeIntegrity Operational | App Control policy active | Confirms WDAC/driver blocklist deployment. Path: Applications and Services Logs > Microsoft > Windows > CodeIntegrity > Operational. ([Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/microsoft-recommended-driver-block-rules)) |

**Query for past Rentdrv2/PoisonX driver loads (Sysmon):**
```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=6} |
  Where-Object { $_.Message -match 'rentdrv2|PoisonX|F8284233' }
```

**Query for suspicious kernel service creation:**
```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Id=7045} |
  Where-Object { $_.Message -match 'kernel|rentdrv2|PoisonX' }
```

**EDR silence detection:** If your EDR agent stops reporting telemetry, that silence is itself an indicator. Centralized monitoring should alert when any endpoint goes quiet unexpectedly.

### Detection rules

**Sigma (SigmaHQ):**
- "Vulnerable Driver Load" (ID: `7aaaf4b8-e47c-4295-92ee-6ed40a6f60c8`) matches against 1,200+ MD5/SHA1 hashes from LOLDrivers. MITRE ATT&CK: T1543.003, T1068. [Source: detection.fyi](https://detection.fyi/sigmahq/sigma/windows/driver_load/driver_load_win_vuln_drivers/)
- "Vulnerable Driver Load By Name" (ID: `72cd00d6-490c-4650-86ff-1d11f491daa1`) matches against 296 driver filenames. Note: Rentdrv2 and PoisonX are **not currently in this rule's name list** and would need to be added manually. [Source: detection.fyi](https://detection.fyi/sigmahq/sigma/windows/driver_load/driver_load_win_vuln_drivers_names/)

**Splunk:**
- "Windows Vulnerable Driver Loaded" (ID: `a2b1f1ef-221f-4187-b2a4-d4b08ec745f4`) uses Sysmon EventCode=6 cross-referenced against the LOLDrivers lookup table. [Source: Splunk Security Content](https://research.splunk.com/endpoint/a2b1f1ef-221f-4187-b2a4-d4b08ec745f4/)

**LOLDrivers detection artifacts** (auto-generated from the driver catalog, covers both Rentdrv2 and PoisonX):
- YARA rules: `detections/yara/yara-rules_vuln_drivers_strict.yar`
- Sigma rules: `detections/sigma/` directory
- Sysmon config: `detections/sysmon/sysmon_config_vulnerable_hashes_block.xml`
- [Source: loldrivers.io](https://www.loldrivers.io/)

**Microsoft Defender ASR rule:**
- "Block abuse of exploited vulnerable signed drivers"
- GUID: `56a863a9-875e-4185-98a7-b882c64b5ce5`
- Note: this rule prevents an application from writing a vulnerable signed driver to disk. It does not block a driver that is already present on the system from loading. Use in combination with the vulnerable driver blocklist and HVCI, not as a standalone control.
- [Source: Microsoft Learn](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-reference)

### Forensic triage: has this already happened?

If you suspect this driver was used in your environment, check for these artifacts:

**Registry:**
- `HKLM\SYSTEM\CurrentControlSet\Services\<driver_name>` created when using `sc create` to register the driver
- Note: the [PoisonKiller_bof](https://github.com/Muz1K1zuM/PoisonKiller_bof) (Cobalt Strike BOF) variant creates a randomly-named 8-character service key and deletes it after use, making registry evidence transient

**File system:**
- Known drop path observed in the wild: `C:\AU_Data\1721289943.sys` (renamed rentdrv2.sys, observed by CrowdStrike in a real intrusion) ([CrowdStrike blog](https://www.crowdstrike.com/en-us/blog/falcon-prevents-vulnerable-driver-attacks-real-world-intrusion/))
- Common drop locations: `C:\Windows\Temp\`, `C:\Windows\System32\drivers\`, or user-writable directories
- Search for files matching any of the SHA256 hashes listed above

**Device path:**
- `\\.\{F8284233-48F4-4680-ADDD-F8284233}` is the symbolic link for the PoisonX device. Any reference to this GUID in logs, memory dumps, or forensic images is a strong indicator.

### Real-world threat actor usage

This is not theoretical. Active ransomware groups use this driver in production attacks:

- **Agonizing Serpens (Agrius)**: Iranian APT, used Rentdrv2 via custom loader `drvIX.exe` against Israeli higher education and tech sector targets since October 2023 ([Palo Alto Unit 42](https://unit42.paloaltonetworks.com/agonizing-serpens-targets-israeli-tech-higher-ed-sectors/))
- **DragonForce ransomware**: Uses Rentdrv2.sys (BadRentdrv2) as one of two BYOVD backends. Config field `use_sys=2` selects rentdrv2.sys. [Source: S2W Medium](https://medium.com/s2wblog/detailed-analysis-of-dragonforce-ransomware-25d1a91a4509), [Acronis TRU](https://www.acronis.com/en/tru/posts/the-dragonforce-cartel-scattered-spider-at-the-gate/)

---

## Confidence Assessment

| Claim | Confidence | Basis |
|-------|------------|-------|
| Bit field decoding of 0x22E010 | **High** | Mathematical decomposition per Microsoft CTL_CODE specification |
| Association with Rentdrv2/PoisonX driver | **High** | Multiple independent sources: CVE database, Unit 42 report, researcher blog, GitHub PoCs |
| Process kill mechanism (ZwOpenProcess/ZwTerminateProcess) | **High** | Confirmed by independent reverse engineering (Medium blog) and corroborated by PoC source code (GitHub) |
| CVE-2023-44976 details | **High** | Sourced directly from NVD |
| Agonizing Serpens attribution | **High** | Sourced from Palo Alto Unit 42 threat intelligence report |
| CVSS score being understated | **Medium** | Analytical judgment; the base score follows CVSS methodology but doesn't capture chained attack impact |
| 0/71 VT detection rate | **Medium** | Reported as 0/71 at the time the PoC author documented the sample; detection rates are inherently unstable and may have changed |

---

## Sources

### Primary Research

- Abudagga, Jehad. ["Reverse Engineering a 0day used Against CrowdStrike EDR."](https://medium.com/@jehadbudagga/reverse-engineering-a-0day-used-against-crowdstrike-edr-a5ea1fbe3fd4) Medium, April 2026.
- ["Researcher Reverse Engineered 0-Day Used to Disable CrowdStrike EDR."](https://cybersecuritynews.com/0-day-disable-crowdstrike-edr/) Cybersecurity News, April 14, 2026.

### Threat Intelligence

- Palo Alto Networks Unit 42. ["Agonizing Serpens (Aka Agrius) Targeting the Israeli Higher Education and Tech Sectors."](https://unit42.paloaltonetworks.com/agonizing-serpens-targets-israeli-tech-higher-ed-sectors/)
- CrowdStrike. ["Falcon Prevents Vulnerable Driver Attacks in Real-World Intrusion."](https://www.crowdstrike.com/en-us/blog/falcon-prevents-vulnerable-driver-attacks-real-world-intrusion/)
- S2W. ["Detailed Analysis of DragonForce Ransomware."](https://medium.com/s2wblog/detailed-analysis-of-dragonforce-ransomware-25d1a91a4509) Medium.
- Acronis TRU. ["The DragonForce Cartel: Scattered Spider at the Gate."](https://www.acronis.com/en/tru/posts/the-dragonforce-cartel-scattered-spider-at-the-gate/)

### Driver Databases

- LOLDrivers. ["Rentdrv2 entry."](https://www.loldrivers.io/drivers/afb8bb46-1d13-407d-9866-1daa7c82ca63/)
- LOLDrivers. ["PoisonX entry."](https://www.loldrivers.io/drivers/fc3467c3-6109-447d-b438-7a4276c3d8e5/)
- [keowu/BadRentdrv2.](https://github.com/keowu/BadRentdrv2) GitHub.

### Detection Rules

- SigmaHQ. ["Vulnerable Driver Load."](https://detection.fyi/sigmahq/sigma/windows/driver_load/driver_load_win_vuln_drivers/)
- Splunk. ["Windows Vulnerable Driver Loaded."](https://research.splunk.com/endpoint/a2b1f1ef-221f-4187-b2a4-d4b08ec745f4/)
- Microsoft Learn. ["Attack Surface Reduction Rules Reference."](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-reference)
- Microsoft Learn. ["Microsoft Recommended Driver Block Rules."](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/design/microsoft-recommended-driver-block-rules)
- HotCakeX. ["Harden-Windows-Security WDAC Notes."](https://github.com/HotCakeX/Harden-Windows-Security/wiki/WDAC-Notes)

### Vulnerability Databases

- NIST NVD. ["CVE-2023-44976 Detail."](https://nvd.nist.gov/vuln/detail/CVE-2023-44976)
- Feedly. ["CVE-2023-44976 - Exploits & Severity."](https://feedly.com/cve/CVE-2023-44976)
- CISA. ["Vulnerability Summary for the Week of July 28, 2025."](https://www.cisa.gov/news-events/bulletins/sb25-216)

### Proof of Concept / Source Code

- [BlackSnufkin/BYOVD](https://github.com/BlackSnufkin/BYOVD/blob/main/PoisonX-Killer/src/main.rs) - PoisonX-Killer (Rust implementation).
- [Nekr0w/poisonkiller](https://github.com/Nekr0w/poisonkiller) - PoisonKiller PoC.
- [j3h4ck/PoisonKiller](https://github.com/j3h4ck/PoisonKiller) - Fork/variant.
- [Muz1K1zuM/PoisonKiller_bof](https://github.com/Muz1K1zuM/PoisonKiller_bof) - Cobalt Strike BOF version.

### Microsoft Documentation

- Microsoft Learn. ["Defining I/O Control Codes."](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/defining-i-o-control-codes)
- Microsoft Learn. ["!ioctldecode."](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-ioctldecode)
- Microsoft Learn. ["Security Issues for I/O Control Codes."](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/security-issues-for-i-o-control-codes)

### Tools

- OSR IOCTL Decoder (online). Referenced but not fetched.
- z4ksec. ["IoctlHunter Release (v0.2)."](https://z4ksec.github.io/posts/ioctlhunter-release-v0.2/)
- daaximus. ["Most IOCTLs mapped to their code names."](https://gist.github.com/daaximus/e813aa52980fc2a97a8a8a1082338de4) GitHub Gist.

---

## How This Report Was Made

This report was generated using Claude Code (Anthropic) as a research and drafting tool. The author directed the research scope, reviewed all claims for accuracy, and applied editorial judgment throughout. All factual claims are grounded in cited sources that were fetched and verified during the research process.

**Disclaimer:** This report is provided for informational and defensive security purposes only. While every effort has been made to ensure accuracy, security research involves rapidly evolving information. Claims are tagged with confidence levels where appropriate. Readers should independently verify critical findings before making security decisions. The author is not responsible for actions taken based on this report. Detection rates, blocklist status, and threat actor activity are time-sensitive and may have changed since publication.
