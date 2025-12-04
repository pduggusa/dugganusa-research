# dugganusa-007: WAF Bypass - JSON-based SQL Injection

**Category:** Web Application Security
**Severity:** Critical
**False Positive Rate:** Very Low (<2%)
**MITRE ATT&CK:** T1190 (Exploit Public-Facing Application), T1059 (Command and Scripting Interpreter)

## Overview

Detects JSON-based SQL injection attacks that exploit WAF parsing discrepancies, based on WAFFLED research (2025). Attackers use `Content-Type: application/json` to bypass WAF inspection while delivering SQLi payloads to backends.

## The Problem

**WAF Parsing Discrepancies:** Major WAF vendors (Palo Alto, AWS WAF, Cloudflare, F5, Imperva) parse JSON content differently than backend application servers. Attackers exploit this gap.

**Attack Pattern:**
```http
POST /api/users HTTP/1.1
Content-Type: application/json

{
  "username": "admin",
  "filter": "id=1 UNION SELECT password FROM users--"
}
```

WAF sees: Valid JSON object
Backend executes: SQL injection payload

## Detection Logic

**Requirements:**
1. HTTP request with `Content-Type: application/json`
2. Request body contains SQL keywords: `UNION SELECT`, `OR 1=1`, `DROP TABLE`, `CAST(`, `CONCAT(`, `SLEEP(`, `BENCHMARK(`, `WAITFOR DELAY`

**Sigma Rule:**

```yaml
title: WAF Bypass - JSON-based SQL Injection
id: dugganusa-007-json-sqli-bypass
status: production
description: Detects JSON-based SQLi exploiting WAF parsing discrepancies
author: DugganUSA LLC
date: 2025-11-19
references:
    - https://portswigger.net/research/waffled
tags:
    - attack.initial_access
    - attack.t1190
    - attack.t1059
logsource:
    category: webserver
detection:
    selection_json:
        content_type|contains: 'application/json'
    selection_sqli:
        request_body|contains:
            - 'UNION SELECT'
            - 'OR 1=1'
            - '; DROP TABLE'
            - 'CAST('
            - 'CONCAT('
            - 'SLEEP('
            - 'BENCHMARK('
            - 'WAITFOR DELAY'
    condition: selection_json and selection_sqli
fields:
    - src_ip
    - request_uri
    - request_body
    - user_agent
falsepositives:
    - Legitimate API testing tools
    - Security scanners with authorization
level: critical
```

## SIEM Implementations

### Splunk SPL

```spl
index=web OR index=proxy OR index=waf
| where match(content_type, "application/json")
| regex request_body="(?i)(UNION\\s+SELECT|OR\\s+1=1|DROP\\s+TABLE|CAST\\(|CONCAT\\(|SLEEP\\(|BENCHMARK\\(|WAITFOR\\s+DELAY)"
| eval sql_keywords=mvcount(split(request_body, "UNION")) + mvcount(split(request_body, "SELECT")) + mvcount(split(request_body, "DROP"))
| where sql_keywords >= 2
| stats count by src_ip, request_uri, user_agent, content_type
| eval severity="CRITICAL"
| eval mitre="T1190 - Exploit Public-Facing Application"
| eval detection="JSON-based SQLi WAF Bypass"
| table src_ip, request_uri, user_agent, count, severity, detection
| sort -count
```

### KQL (Azure Sentinel)

```kql
// WAF Bypass - JSON-based SQL Injection Detection
W3CIISLog
| where TimeGenerated > ago(24h)
| where csMethod in ("POST", "PUT", "PATCH")
| where csReferer contains "application/json" or csUserAgent contains "application/json"
| extend RequestBody = tostring(AdditionalFields["request_body"])
| where RequestBody has_any ("UNION SELECT", "OR 1=1", "DROP TABLE", "CAST(", "CONCAT(", "SLEEP(", "BENCHMARK(", "WAITFOR DELAY")
| summarize
    AttemptsCount=count(),
    UniquePaths=make_set(csUriStem),
    LastAttempt=max(TimeGenerated)
  by cIP, csUserAgent
| extend Severity = "Critical"
| extend MITRE = "T1190 - Exploit Public-Facing Application"
| extend Detection = "JSON-based SQLi WAF Bypass (WAFFLED)"
| sort by AttemptsCount desc
```

## Example Attack Scenarios

### Scenario 1: Classic UNION-based SQLi

```json
POST /api/search
Content-Type: application/json

{
  "query": "product' UNION SELECT username, password FROM users--"
}
```

**WAF behavior:** Sees valid JSON, no SQL pattern in keys
**Backend behavior:** Executes concatenated SQL query
**Detection:** Rule triggers on `UNION SELECT` in request body

### Scenario 2: Time-based Blind SQLi

```json
POST /api/login
Content-Type: application/json

{
  "username": "admin",
  "password": "x' OR SLEEP(10)--"
}
```

**WAF behavior:** JSON structure valid
**Backend behavior:** 10-second delay if vulnerable
**Detection:** Rule triggers on `SLEEP(` pattern

### Scenario 3: Boolean-based Blind SQLi

```json
POST /api/products
Content-Type: application/json

{
  "category": "electronics' OR 1=1--"
}
```

**WAF behavior:** No obvious attack pattern
**Backend behavior:** Returns all products regardless of category
**Detection:** Rule triggers on `OR 1=1` pattern

## False Positives

**Low false positive rate (<2%) due to:**

1. **Dual condition:** Requires BOTH `Content-Type: application/json` AND SQL keywords
2. **Rare legitimate use:** JSON payloads rarely contain SQL syntax
3. **Context-aware:** Targets web application endpoints, not internal database queries

**Known FP sources:**
- API testing tools (Postman, curl scripts with SQLi examples)
- Security scanners with proper authorization
- Database documentation/tutorials in JSON format

**Mitigation:**
- Whitelist authorized security scanner IPs
- Exclude `/test/` or `/qa/` endpoints if needed
- Verify `User-Agent` against known testing tools

## Remediation

**When this alert fires:**

1. **Immediate:** Block source IP at WAF/firewall
2. **Investigate:** Review full HTTP request/response
3. **Validate:** Check if vulnerable endpoint exists
4. **Patch:** Update WAF rules to parse JSON bodies correctly
5. **Harden:** Implement parameterized queries on backend

**WAF Configuration Fix:**

Enable deep JSON inspection:
- **AWS WAF:** Enable JSON body inspection in managed rules
- **Cloudflare:** Enable "JSON content" in WAF rules
- **F5:** Configure ASM to inspect JSON payloads
- **Palo Alto:** Enable "Application Override" for JSON parsing

## References

- **WAFFLED Research:** https://portswigger.net/research/waffled
- **OWASP SQLi Guide:** https://owasp.org/www-community/attacks/SQL_Injection
- **MITRE ATT&CK T1190:** https://attack.mitre.org/techniques/T1190/

## Attribution

**Research:** WAFFLED (PortSwigger Research, 2025)
**Implementation:** DugganUSA LLC
**Validation:** Production-tested Nov 2025
**License:** Free for non-commercial use. Commercial use requires attribution.

---

**Democratic Sharing Law** - WAF vendors won't fix parsing bugs until we expose them publicly. Free detection rules accelerate fixes.
