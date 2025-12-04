# dugganusa-009: XSS Bypass via Encoding Tricks

**Category:** Web Application Security
**Severity:** High
**False Positive Rate:** Medium (10-12%)
**MITRE ATT&CK:** T1190 (Exploit Public-Facing Application), T1059.007 (JavaScript)

## Overview

Detects XSS attacks using encoding tricks (HTML entities, Unicode escapes, Base64, double encoding) to bypass WAF filters. Attackers exploit the fact that WAFs often fail to decode multiple encoding layers before pattern matching.

## The Problem

**Multi-Layer Encoding Bypass:** WAFs typically decode URLs once, but attackers can:
1. Double-encode payloads (`%253Cscript` = `%3Cscript` = `<script`)
2. Use HTML entities (`&#x3C;script` = `<script`)
3. Mix Unicode escapes (`\u003cscript` = `<script`)
4. Base64-encode JavaScript (`atob('YWxlcnQoMSk=')` = `alert(1)`)

**WAF Limitation:** Most WAFs decode once, miss second layer, pass malicious payload to backend.

## Detection Logic

**Triggers on encoded XSS patterns:**

- **HTML Entities:** `&#x` (hex entities for `<`, `/`, `>`)
- **URL Encoding:** `%3Cscript`, `%3Bjavascript`
- **Unicode Escapes:** `\u003c`, `\x3c`
- **JavaScript Encoding:** `atob(`, `fromCharCode(`, `eval(atob`, `String.fromCharCode`
- **Double Encoding:** `%253C` (`%` encoded as `%25`)

**Sigma Rule:**

```yaml
title: XSS Bypass via Encoding Tricks
id: dugganusa-009-xss-encoding-bypass
status: production
description: Encoded XSS payloads bypassing WAF filters
author: DugganUSA LLC
date: 2025-11-19
tags:
    - attack.initial_access
    - attack.t1190
    - attack.t1059.007
logsource:
    category: webserver
detection:
    selection_encoded_xss:
        request_uri|contains:
            - '&#x'
            - '%3Cscript'
            - '%3Bjavascript'
            - '\\u003c'
            - '\\x3c'
            - 'atob('
            - 'fromCharCode('
            - 'eval(atob'
            - 'String.fromCharCode'
    selection_double_encoding:
        request_uri|contains:
            - '%253C'
            - '%2527'
            - '%252F'
    condition: selection_encoded_xss or selection_double_encoding
fields:
    - src_ip
    - request_uri
    - user_agent
falsepositives:
    - Legitimate applications using URL encoding
    - Email clients encoding links
level: high
```

## SIEM Implementations

### Splunk SPL

```spl
index=web OR index=proxy
| where match(request_uri, "(?i)(&#x|%3Cscript|%3Bjavascript|\\\\u003c|\\\\x3c|atob\\(|fromCharCode\\(|eval\\(atob|String\\.fromCharCode|%253C|%2527|%252F)")
| eval encoding_type=case(
    match(request_uri, "&#x"), "HTML Entity Encoding",
    match(request_uri, "%253C"), "Double URL Encoding",
    match(request_uri, "\\\\u003c"), "Unicode Escape",
    match(request_uri, "atob\\(|fromCharCode"), "JavaScript Encoding",
    1=1, "Mixed Encoding"
  )
| eval decoded_uri=urldecode(urldecode(request_uri))
| where match(decoded_uri, "(?i)(<script|javascript:|onerror=|onload=)")
| stats count by src_ip, encoding_type, request_uri, user_agent
| eval severity="HIGH"
| eval mitre="T1190 + T1059.007"
| eval detection="XSS Encoding Bypass"
| table src_ip, encoding_type, request_uri, user_agent, count, severity
| sort -count
```

### KQL (Azure Sentinel)

```kql
// XSS Bypass via Encoding Tricks
W3CIISLog
| where TimeGenerated > ago(24h)
| where csUriQuery has_any ("&#x", "%3Cscript", "%3Bjavascript", "\\u003c", "\\x3c", "atob(", "fromCharCode(", "eval(atob", "String.fromCharCode", "%253C", "%2527", "%252F")
| extend EncodingType = case(
    csUriQuery has "&#x", "HTML Entity Encoding",
    csUriQuery has "%253C" or csUriQuery has "%2527", "Double URL Encoding",
    csUriQuery has "\\u003c" or csUriQuery has "\\x3c", "Unicode Escape",
    csUriQuery has "atob(" or csUriQuery has "fromCharCode", "JavaScript Encoding",
    "Mixed Encoding"
  )
| extend DecodedURI = url_decode(url_decode(csUriQuery))
| where DecodedURI has_any ("<script", "javascript:", "onerror=", "onload=")
| summarize Count=count() by cIP, EncodingType, csUriQuery, csUserAgent
| extend Severity = "High"
| extend MITRE = "T1190 + T1059.007"
| extend Detection = "XSS Encoding Bypass"
| sort by Count desc
```

## Example Attack Scenarios

### Scenario 1: HTML Entity Encoding

```http
GET /search?q=&#x3C;script&#x3E;alert(1)&#x3C;/script&#x3E; HTTP/1.1
```

**Decodes to:** `<script>alert(1)</script>`

**Why WAF misses:** WAF checks for literal `<script>`, doesn't decode HTML entities

### Scenario 2: Double URL Encoding

```http
GET /profile?name=%253Cscript%253Ealert(document.cookie)%253C%252Fscript%253E HTTP/1.1
```

**First decode:** `%3Cscript%3Ealert(document.cookie)%3C%2Fscript%3E`
**Second decode:** `<script>alert(document.cookie)</script>`

**Why WAF misses:** WAF decodes once, sees `%3Cscript` (still encoded), passes it through

### Scenario 3: Unicode Escape

```http
GET /comment?text=\u003cscript\u003ealert(1)\u003c/script\u003e HTTP/1.1
```

**JavaScript interprets:** `<script>alert(1)</script>`

**Why WAF misses:** WAF doesn't execute JavaScript, doesn't interpret `\u003c` as `<`

### Scenario 4: Base64 + JavaScript Encoding

```http
GET /api/log?msg=<img src=x onerror="eval(atob('YWxlcnQoZG9jdW1lbnQuY29va2llKQ=='))"> HTTP/1.1
```

**Base64 decodes to:** `alert(document.cookie)`

**Why WAF misses:** WAF sees `atob()` call but doesn't execute it, payload hidden in Base64

### Scenario 5: Mixed Encoding (HTML + Unicode)

```http
GET /search?q=&#x3C;img src=x onerror=\u0061\u006c\u0065\u0072\u0074(1)&#x3E; HTTP/1.1
```

**Decodes to:** `<img src=x onerror=alert(1)>`

**Why WAF misses:** Multiple encoding layers confuse pattern matching

## False Positives

**Moderate false positive rate (10-12%) due to:**

1. **Legitimate URL Encoding:** Normal characters encoded in URLs
2. **Email Clients:** Auto-encode links when forwarding
3. **Analytics Tracking:** URL parameters with encoded data

**Common FP Examples:**
- `?utm_source=%3Cfacebook%3E` (legitimate Facebook tracking)
- `?q=%E2%9C%93` (checkmark emoji URL-encoded)
- `?redirect=https%3A%2F%2Fexample.com` (legitimate redirect)

**Mitigation:**
- Verify decoded payload contains actual XSS (`<script`, `javascript:`, event handlers)
- Exclude known legitimate query parameters (utm_*, fbclid, gclid)
- Check if payload reflects in HTTP response (stored XSS vs failed attempt)

## Remediation

**Immediate Response:**

1. **Block IP:** Add source to blocklist
2. **Decode & Inspect:** Fully decode request, check if XSS succeeded
3. **Review CSP:** Ensure Content Security Policy blocks inline scripts
4. **WAF Update:** Enable recursive decoding in WAF rules

**Long-term Hardening:**

```javascript
// Example: Recursive decoding + XSS sanitization (Node.js)
function sanitizeInput(input) {
  let decoded = input;
  let previous = '';

  // Recursive decode (max 3 iterations)
  for (let i = 0; i < 3 && decoded !== previous; i++) {
    previous = decoded;
    decoded = decodeURIComponent(decoded);
  }

  // Check for XSS patterns after full decode
  const xssPatterns = /<script|javascript:|onerror=|onload=/i;
  if (xssPatterns.test(decoded)) {
    throw new Error('XSS detected');
  }

  return decoded;
}
```

**WAF Configuration:**
- **AWS WAF:** Enable "Transformation" rules (URL_DECODE, HTML_ENTITY_DECODE)
- **Cloudflare:** Set decoding depth to 3 iterations
- **ModSecurity:** `SecRule REQUEST_URI "@rx <script" "phase:2,t:urlDecodeUni,t:htmlEntityDecode,deny"`

**Browser Defense (CSP):**

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self';">
```

## References

- **OWASP XSS Filter Evasion:** https://owasp.org/www-community/xss-filter-evasion-cheatsheet
- **PortSwigger XSS Labs:** https://portswigger.net/web-security/cross-site-scripting
- **MITRE ATT&CK T1059.007:** https://attack.mitre.org/techniques/T1059/007/

## Attribution

**Research:** OWASP Testing Guide 2025 + PortSwigger Research
**Implementation:** DugganUSA LLC
**License:** Free for non-commercial use. Commercial use requires attribution.

---

**Democratic Sharing Law** - WAFs that don't recursively decode are lying to you. Free detection rules expose the truth.
