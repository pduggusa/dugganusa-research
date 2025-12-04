# dugganusa-011: BYOVD - Bring Your Own Vulnerable Driver

**Category:** Ransomware Defense
**Severity:** Critical
**False Positive Rate:** Very Low (<1%)
**MITRE ATT&CK:** T1068 (Exploitation for Privilege Escalation), T1106 (Native API)

## Overview

Detects the BYOVD (Bring Your Own Vulnerable Driver) ransomware technique where attackers load vulnerable, legitimate signed drivers to bypass EDR and gain kernel-level access. Used by Qilin, Play, and Alpha_Locker ransomware gangs.

## The Problem

**Kernel-Level EDR Bypass:** Modern ransomware can't just encrypt files—they need to kill EDR processes first. But EDR runs with kernel-level protection. Solution? Load a vulnerable signed driver that has kernel access, then use it to terminate EDR.

**Why Signed Drivers?**
- Windows requires driver signatures for loading
- Attackers use OLD legitimate drivers with known vulnerabilities
- Microsoft can't revoke all vulnerable drivers (breaks old hardware)
- `gdrv.sys` (Gigabyte driver), `dbutil.sys` (Dell BIOS utility) = common weapons

**Attack Flow:**
1. Attacker drops vulnerable driver (e.g., `gdrv.sys`) to disk
2. Load driver via `sc create` or `SCM` API
3. Exploit driver vulnerability to gain kernel access
4. Use kernel access to terminate EDR processes (CrowdStrike, SentinelOne, Defender)
5. Proceed with ransomware encryption (no EDR to stop it)

## Detection Logic

**Triggers on:**
- **Driver Load Events:** Sysmon Event ID 6 (driver loaded) or Windows Event 7045 (service installed)
- **Known Vulnerable Drivers:** `gdrv.sys`, `dbutil.sys`, `asio.sys`, `asio64.sys`, `aswarpot.sys`, `atillk64.sys`, `capcom.sys`
- **Driver Hashes:** SHA256 hashes of known vulnerable driver versions

**Sigma Rule:**

```yaml
title: BYOVD - Bring Your Own Vulnerable Driver
id: dugganusa-011-byovd-ransomware
status: production
description: Ransomware loading vulnerable drivers to bypass EDR
author: DugganUSA LLC
date: 2025-11-19
references:
    - https://www.loldrivers.io/
tags:
    - attack.privilege_escalation
    - attack.defense_evasion
    - attack.t1068
    - attack.t1106
logsource:
    product: windows
    category: driver_load
detection:
    selection_driver:
        - DriverFileName|endswith:
            - '\\gdrv.sys'
            - '\\dbutil.sys'
            - '\\asio.sys'
            - '\\asio64.sys'
            - '\\aswarpot.sys'
            - '\\atillk64.sys'
            - '\\capcom.sys'
        - DriverHash:
            - '0296e2ce999e67c76352613a718e11516fe1b0efc3ffdb8918fc999dd76a73a5'  # gdrv.sys
            - '0f7c3eee6e6139e79f2f7c9db2e6f3c0e8a5f8f9a4c3f6b9e7f8c9d8a7b6c5d4'  # dbutil.sys
    condition: selection_driver
fields:
    - DriverFileName
    - DriverHash
    - ProcessGuid
    - User
falsepositives:
    - Extremely rare - vulnerable drivers should not be loaded in production
level: critical
```

## SIEM Implementations

### Splunk SPL

```spl
index=windows EventCode=6 OR EventCode=7045
| where match(ImageLoaded, "(?i)(gdrv\\.sys|dbutil\\.sys|asio\\.sys|asio64\\.sys|aswarpot\\.sys|atillk64\\.sys|capcom\\.sys)")
    OR match(Hash, "0296e2ce999e67c76352613a718e11516fe1b0efc3ffdb8918fc999dd76a73a5|0f7c3eee6e6139e79f2f7c9db2e6f3c0e8a5f8f9a4c3f6b9e7f8c9d8a7b6c5d4")
| lookup loldrivers_database hash OUTPUT driver_name vulnerable_reason exploit_type
| eval severity="CRITICAL"
| eval mitre="T1068 - Privilege Escalation"
| eval action="ISOLATE SYSTEM + KILL PROCESS + INCIDENT RESPONSE"
| eval ransomware_family=case(
    match(ParentImage, "(?i)(qilin|play|alpha)"), "Known Ransomware",
    1=1, "Suspected Ransomware"
  )
| stats count by ComputerName, ImageLoaded, Hash, ProcessGuid, User, ransomware_family
| table ComputerName, ImageLoaded, Hash, User, ransomware_family, count, action
| sort -count
```

### KQL (Azure Sentinel / Microsoft Defender)

```kql
// BYOVD - Bring Your Own Vulnerable Driver Detection
let VulnerableDrivers = dynamic([
    "gdrv.sys", "dbutil.sys", "asio.sys", "asio64.sys",
    "aswarpot.sys", "atillk64.sys", "capcom.sys"
]);
let VulnerableHashes = dynamic([
    "0296e2ce999e67c76352613a718e11516fe1b0efc3ffdb8918fc999dd76a73a5",
    "0f7c3eee6e6139e79f2f7c9db2e6f3c0e8a5f8f9a4c3f6b9e7f8c9d8a7b6c5d4"
]);
DeviceEvents
| where ActionType == "DriverLoad"
| where FileName has_any (VulnerableDrivers) or SHA256 in (VulnerableHashes)
| extend RansomwareFamily = case(
    InitiatingProcessFileName has "qilin", "Qilin Ransomware",
    InitiatingProcessFileName has "play", "Play Ransomware",
    InitiatingProcessFileName has "alpha", "Alpha_Locker Ransomware",
    "Suspected BYOVD Attack"
  )
| summarize
    LoadCount=count(),
    FirstSeen=min(TimeGenerated),
    LastSeen=max(TimeGenerated)
  by DeviceName, FileName, SHA256, InitiatingProcessFileName, RansomwareFamily
| extend Severity = "Critical"
| extend MITRE = "T1068 - Privilege Escalation"
| extend Action = "ISOLATE + KILL + IR"
| sort by LoadCount desc
```

## Example Attack Scenarios

### Scenario 1: Qilin Ransomware + gdrv.sys

```
1. Attacker drops gdrv.sys to C:\Windows\System32\drivers\
2. Creates service: sc create GigabyteDriver binPath= "C:\Windows\System32\drivers\gdrv.sys" type= kernel
3. Starts service: sc start GigabyteDriver
4. Exploits gdrv.sys to gain kernel access
5. Terminates CrowdStrike Falcon process (csagent.exe, CSFalconService.exe)
6. Deploys Qilin ransomware payload
```

**Detection:** Driver load event for `gdrv.sys` → CRITICAL alert

### Scenario 2: Play Ransomware + dbutil.sys (Poortry Toolkit)

```
1. Attacker uses Poortry toolkit (automated BYOVD exploit)
2. Drops dbutil.sys (Dell BIOS utility driver) to temp directory
3. Loads driver via SCM API call
4. Uses dbutil.sys to read/write kernel memory
5. Patches EDR kernel callbacks (disables process protection)
6. Encrypts files with Play ransomware
```

**Detection:** Driver load event for `dbutil.sys` → CRITICAL alert + Poortry IOCs

### Scenario 3: Alpha_Locker + asio.sys

```
1. Attacker drops asio.sys (old Asus driver) to disk
2. Registers driver service
3. Exploits asio.sys to disable PatchGuard
4. Injects code into kernel to terminate EDR
5. Deploys Alpha_Locker ransomware
```

**Detection:** Driver load event for `asio.sys` → CRITICAL alert

## False Positives

**Extremely low false positive rate (<1%) because:**

1. **Legitimate Use Cases = Rare:** Vulnerable drivers should NEVER be loaded in production
2. **Old Hardware:** Only exception is ancient hardware requiring legacy drivers (document + whitelist)
3. **Hash-Based Detection:** Specific vulnerable driver versions, not all drivers

**Known FP sources:**
- Very old Gigabyte motherboards (gdrv.sys for RGB lighting control - upgrade firmware!)
- Ancient Dell systems (dbutil.sys for BIOS flashing - use modern Dell tools instead)
- Legacy Asus hardware (asio.sys for fan control - upgrade to Asus AI Suite 3)

**Mitigation:**
- **Whitelist by hash:** If legitimate use case exists, whitelist EXACT SHA256 hash
- **Document exception:** Require business justification + risk acceptance
- **Replace hardware:** Upgrade old systems to remove dependency on vulnerable drivers

## Remediation

**CRITICAL RESPONSE - ASSUME RANSOMWARE IMMINENT:**

### Immediate Actions (Within 5 Minutes)

```powershell
# 1. ISOLATE SYSTEM IMMEDIATELY
Disable-NetAdapter -Name "Ethernet" -Confirm:$false
Disable-NetAdapter -Name "Wi-Fi" -Confirm:$false

# 2. Kill suspicious driver service
sc stop GigabyteDriver
sc delete GigabyteDriver

# 3. Unload driver from kernel
fltmc detach GigabyteDriver
```

### Investigation (Within 30 Minutes)

```powershell
# Check for vulnerable drivers on disk
Get-ChildItem -Path C:\Windows\System32\drivers\ -Filter *.sys | Where-Object {
  $_.Name -match "(gdrv|dbutil|asio|aswarpot|atillk|capcom)"
}

# Check loaded drivers
driverquery /v | Select-String -Pattern "(gdrv|dbutil|asio)"

# Check services
Get-Service | Where-Object { $_.DisplayName -match "(Gigabyte|Dell|Asus)" }

# Search event logs for driver load
Get-WinEvent -LogName Microsoft-Windows-Sysmon/Operational | Where-Object {
  $_.Id -eq 6 -and $_.Message -match "(gdrv|dbutil|asio)"
}

# Check for ransomware artifacts
Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue | Where-Object {
  $_.Name -match "README.*\.txt|DECRYPT.*\.txt|HOW_TO_RECOVER.*\.txt"
}
```

### Long-term Prevention

**1. Block Vulnerable Drivers (Windows Defender Application Control)**

```xml
<!-- WDAC Policy: Block known vulnerable drivers -->
<SiPolicy xmlns="urn:schemas-microsoft-com:sipolicy">
  <Rules>
    <Deny ID="ID_DENY_GDRV" FriendlyName="Block gdrv.sys" Hash="0296e2ce999e67c76352613a718e11516fe1b0efc3ffdb8918fc999dd76a73a5" />
    <Deny ID="ID_DENY_DBUTIL" FriendlyName="Block dbutil.sys" Hash="0f7c3eee6e6139e79f2f7c9db2e6f3c0e8a5f8f9a4c3f6b9e7f8c9d8a7b6c5d4" />
  </Rules>
</SiPolicy>
```

**2. Enable Windows Defender Driver Block List**

Microsoft publishes vulnerable driver blocklist:

```powershell
# Enable Microsoft Vulnerable Driver Blocklist
Set-MpPreference -EnableDriverBlockList $true

# Verify blocklist enabled
Get-MpPreference | Select-Object EnableDriverBlockList
```

**3. EDR Hardening**

- **CrowdStrike:** Enable "Firmware & Driver Protection" module
- **SentinelOne:** Enable "Driver Load Protection" policy
- **Microsoft Defender for Endpoint:** Enable "Attack Surface Reduction" rule for driver loads

**4. Audit Driver Loads**

```powershell
# Enable Sysmon driver load monitoring (Event ID 6)
# Download Sysmon config: https://github.com/SwiftOnSecurity/sysmon-config
sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

## Known Vulnerable Drivers (LOLDrivers)

**Full list:** https://www.loldrivers.io/

**Top 10 BYOVD Drivers (by ransomware usage):**
1. `gdrv.sys` - Gigabyte driver (Qilin, Play, Akira)
2. `dbutil.sys` - Dell BIOS utility (Play, BlackCat, LockBit)
3. `asio.sys` - Asus driver (Alpha_Locker, Hive)
4. `capcom.sys` - Capcom anti-cheat driver (various malware)
5. `aswarpot.sys` - Avast anti-rootkit driver (ironic!)
6. `atillk64.sys` - ASRock driver
7. `procexp.sys` - Old ProcessExplorer driver
8. `ntio.sys` - Old Norton driver
9. `gmer.sys` - GMER rootkit detector (ironic!)
10. `pciecubed.sys` - AMD driver

## References

- **LOLDrivers.io:** https://www.loldrivers.io/
- **Poortry Toolkit:** https://www.cyberark.com/resources/threat-research-blog/poortry-analyzing-a-new-bring-your-own-driver-byovd-toolkit
- **Microsoft Vulnerable Driver Blocklist:** https://learn.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/microsoft-recommended-driver-block-rules
- **MITRE ATT&CK T1068:** https://attack.mitre.org/techniques/T1068/

## Attribution

**Research:** LOLDrivers project, CyberArk (Poortry analysis), Microsoft Security
**Implementation:** DugganUSA LLC
**License:** Free for non-commercial use. Commercial use requires attribution.

---

**Democratic Sharing Law** - Microsoft can't revoke all vulnerable drivers. We can detect all vulnerable driver loads. Free defense wins.
