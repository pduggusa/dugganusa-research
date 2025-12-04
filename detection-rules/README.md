# Detection Rules

Production-tested detection rules extracted from DugganUSA auto-blocking system.

**Storage:** In-memory library (`/lib/detection-rules.js`)
**Source:** Production auto-blocker + threat intelligence research
**License:** Free for non-commercial use. Commercial use requires attribution.

## Overview

**Philosophy:** "Here's how we detect the assholes - copy this into your SIEM"

Each detection rule includes:
- **Multi-SIEM formats** - Sigma YAML, Splunk SPL, KQL (Azure Sentinel), JSON
- **MITRE ATT&CK mapping** - Adversary techniques covered
- **False positive rates** - Measured from production incidents
- **Source code reference** - Exact line in auto-blocker.js (for original 6 rules)
- **Rationale** - Why we detect this way (evidence-based)

## Available Rules

| Rule ID | Name | Category | Severity | FPR | MITRE |
|---------|------|----------|----------|-----|-------|
| [dugganusa-001](./dugganusa-001.md) | High-Confidence Malicious IP Detection | Network Traffic | High | 5.96% | T1071 |
| [dugganusa-002](./dugganusa-002.md) | VirusTotal Malicious Detection | Endpoint Security | Critical | <1% | T1071 |
| [dugganusa-003](./dugganusa-003.md) | Suspicious ISP Keywords Detection | Threat Hunting | Medium | 10-15% | T1090, T1102 |
| [dugganusa-004](./dugganusa-004.md) | Young Domain Detection (T1568) | DNS Security | High | Medium | T1568 |
| [dugganusa-005](./dugganusa-005.md) | Repeat Offender ISP Pattern | Behavioral Analysis | High | Low | T1071 |
| [dugganusa-006](./dugganusa-006.md) | Cloud Provider Abuse Detection (T1102) | Cloud Security | Medium | Low | T1102 |
| [dugganusa-007](./dugganusa-007.md) | WAF Bypass - JSON-based SQL Injection | Web Application Security | Critical | <2% | T1190, T1059 |
| [dugganusa-008](./dugganusa-008.md) | WAF Bypass - HTTP Header Injection | Web Application Security | High | 5-8% | T1190 |
| [dugganusa-009](./dugganusa-009.md) | XSS Bypass via Encoding Tricks | Web Application Security | High | 10-12% | T1190, T1059.007 |
| [dugganusa-010](./dugganusa-010.md) | CISA KEV Exploitation Attempts | Vulnerability Management | Critical | <1% | T1190 |
| [dugganusa-011](./dugganusa-011.md) | BYOVD - Bring Your Own Vulnerable Driver | Ransomware Defense | Critical | <1% | T1068, T1106 |
| [dugganusa-012](./dugganusa-012.md) | Living-off-the-Land Ransomware Tactics | Ransomware Defense | High | 15-20% | T1218, T1059 |

## Rule Categories

### Network Traffic (2 rules)
- High-Confidence Malicious IP Detection
- Repeat Offender ISP Pattern

### Web Application Security (3 rules)
- WAF Bypass - JSON-based SQL Injection
- WAF Bypass - HTTP Header Injection
- XSS Bypass via Encoding Tricks

### Ransomware Defense (2 rules)
- BYOVD - Bring Your Own Vulnerable Driver
- Living-off-the-Land Ransomware Tactics

### Endpoint Security (1 rule)
- VirusTotal Malicious Detection

### Threat Hunting (1 rule)
- Suspicious ISP Keywords Detection

### DNS Security (1 rule)
- Young Domain Detection (T1568)

### Cloud Security (1 rule)
- Cloud Provider Abuse Detection (T1102)

### Vulnerability Management (1 rule)
- CISA KEV Exploitation Attempts

## Usage

### Via API

```bash
# Get all detection rules
curl https://analytics.dugganusa.com/api/v1/detection-rules

# Get rule by ID
curl https://analytics.dugganusa.com/api/v1/detection-rules/dugganusa-001

# Export in Splunk SPL format
curl https://analytics.dugganusa.com/api/v1/detection-rules/dugganusa-001/export?format=splunk

# Export in KQL format (Azure Sentinel)
curl https://analytics.dugganusa.com/api/v1/detection-rules/dugganusa-001/export?format=kql

# Export in Sigma YAML format
curl https://analytics.dugganusa.com/api/v1/detection-rules/dugganusa-001/export?format=sigma

# Get statistics
curl https://analytics.dugganusa.com/api/v1/detection-rules/stats
```

### Via UI

Navigate to: `https://analytics.dugganusa.com/#detection-rules`

Filter by category → Click rule → Export in your SIEM format

### Via Markdown

Read individual rule files in this directory for human-readable format

## Multi-SIEM Compatibility

**Supported SIEM platforms:**

- **Splunk Enterprise Security** - Import SPL queries directly
- **Microsoft Azure Sentinel** - Import KQL queries
- **Elastic Security** - Convert Sigma YAML to ElasticSearch DSL
- **IBM QRadar, Rapid7, LogRhythm** - Convert Sigma to native formats
- **Any SIEM** - Use Sigma YAML with sigmac converter

**Philosophy:** Zero marginal cost for digital goods. We spent the time building these. Sharing them costs us nothing. Why be a greedy asshole?

## Evidence-Based Detection

These aren't theoretical rules. They come from our live auto-blocking system that has:

- Blocked 1,500+ malicious IPs in 90 days
- Processed data from 7 threat intel sources (AbuseIPDB, VirusTotal, ThreatFox, GreyNoise, IPQualityScore, Cloudflare, MISP)
- Generated 559 STIX 2.1 indicators with full provenance
- Achieved 94.04% precision after expert-curation

**We eat our own dog food.** These rules protect our production infrastructure right now.

## The Paul Galjan Incident

**Why rule thresholds matter:**

On Nov 3, 2025, we blocked Paul Galjan (DARPA/OSD background, 1996-2000, our King/Avi partner) from accessing our site. Abuse score: 0. Reports: 0. **False positive.**

**Root cause:** Threshold too aggressive (abuse score >5)

**Fix:** Raised threshold to >15 (dugganusa-001)

**Result:** False positive rate dropped from 63% to 5.96% (157% precision improvement)

**The Aristocrats Standard:** Admit mistakes, show receipts, thank those wronged, fix publicly.

## Integration with SOC Triad

**Complete threat detection & response platform:**

1. **Detection Rules** (this document) - Alert on known-bad (reactive)
2. **Remediation Playbooks** (`/docs/remediation-playbooks/`) - Respond when alerts fire
3. **Threat Hunt Queries** (`/docs/threat-hunt-queries/`) - Find unknown-bad (proactive)

**Workflow:**
- Detection rule fires → Open remediation playbook → Follow steps → Resolve in 15 minutes
- Hunting query finds threat → Create detection rule for future → Auto-alert on recurrence

## Attribution

Built by: **DugganUSA LLC** (Minnesota)
DARPA validation: **Full Bono methodology** (Paul Galjan, OSD 1996-2000)
Open source: **Democratic Sharing Law** (99.5% public files)

---

**Democratic Sharing Law** - Detection catches known-bad. Sharing catches everyone else's unknown-bad. Together we're unstoppable.
