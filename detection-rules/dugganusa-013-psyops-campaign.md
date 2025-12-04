# Detection Rule: DUGGANUSA-013 - PsyOps Manipulation Campaign

**Status:** Active
**Severity:** Medium-High (varies by score)
**Confidence:** 95% (epistemic humility cap)
**First Seen:** November 21, 2025
**Last Updated:** November 21, 2025

---

## Summary

Detects coordinated psychological manipulation campaigns using the 20-Point NCI Mind Control Detection System. Identifies scams, social engineering attacks, and engineered narratives through pattern recognition of manipulation indicators.

**Philosophy:** "Mind control isn't magic — it's pattern recognition."

---

## MITRE ATT&CK Mapping

### Tactics
- **TA0001:** Initial Access
- **TA0009:** Collection (gathering victim information)
- **TA0043:** Reconnaissance

### Techniques
- **T1566:** Phishing (all sub-techniques)
  - T1566.001: Spearphishing Attachment
  - T1566.002: Spearphishing Link
  - T1566.003: Spearphishing via Service
- **T1598:** Phishing for Information
  - T1598.001: Spearphishing Service
  - T1598.002: Spearphishing Attachment
  - T1598.003: Spearphishing Link
- **T1534:** Internal Spearphishing
- **T1591:** Gather Victim Org Information
- **T1589:** Gather Victim Identity Information
- **T1586:** Compromise Accounts (social media manipulation)

### Sub-Techniques
- Social Engineering (T1656)
- Credentials from Password Stores (T1555)
- Multi-Factor Authentication Request Generation (T1621)

---

## Detection Logic

### Scoring Framework

Each of 20 indicators scores 1-5 points:
- **1** = Not present
- **2** = Weakly present
- **3** = Moderately present
- **4** = Strongly present
- **5** = Overwhelmingly present

**Total Score Interpretation:**
- **0-25:** Low manipulation (neutral content, technical attacks)
- **26-50:** Moderate manipulation (high-pressure sales, targeted phishing)
- **51-75:** Strong manipulation (scam campaigns, coordinated social engineering)
- **76-95:** Overwhelming manipulation (phone scams, credential stuffing, engineered narratives)

**All scores CAPPED at 95%** - Never claim 100% certainty (epistemic humility)

### 20 Indicators

**Category 1: Information Manipulation (1-5)**

1. **Timing Manipulation** - Suspiciously convenient release timing
2. **Emotional Manipulation** - Triggers fear, anger, guilt, shame
3. **Uniform Messaging** - Different sources, identical wording
4. **Missing Information** - Key details or context absent
5. **Simplistic Narrative** - Complex issue reduced to black/white

**Category 2: Emotional Manipulation (6-10)**

6. **Tribal Division** - Divides people into opposing groups
7. **Authority Overload** - Overwhelms with experts/official claims
8. **Urgency / Call to Action** - Forces fast decisions
9. **Novelty Obsession** - Shocking new claims
10. **Financial / Power Gain** - Clear beneficiary if you believe

**Category 3: Cognitive Manipulation (11-15)**

11. **Suppression of Dissent** - Critics shamed or dismissed
12. **No Alternatives Offered** - Only one "solution" presented
13. **Emotional Repetition** - Feelings repeated until accepted
14. **Cherry-Picked Data** - Facts selected for false impression
15. **Logical Fallacies** - Bad logic (bandwagon, false dilemma, strawman)

**Category 4: Technical & Advanced Manipulation (16-20)**

16. **Face-Driven Appeals** - Emotional imagery of people
17. **Manufactured Outrage** - Artificially intensified emotions
18. **Framing Manipulation** - Selective angles/language to steer opinion
19. **Rapid Behavior Shifts** - Sudden or extreme changes demanded
20. **Historical Parallels** - Inappropriate comparisons to extreme events

---

## Implementation

### Threat Intelligence Scoring

```javascript
const { scoreThreat, getSeverityLevel } = require('./lib/psyops-scorer');

const threat = {
  ip: '1.2.3.4',
  isp: 'Evil Corp ISP',
  usageType: 'Residential Proxy',
  abuseScore: 85,
  totalReports: 150,
  vtMalicious: 5,
  isSocialEngineering: true // Key flag for PsyOps scoring
};

const score = scoreThreat(threat);
console.log(`PsyOps Score: ${score}/95 (${getSeverityLevel(score)})`);
```

### Campaign Scoring

```javascript
const { scoreCampaign } = require('./lib/psyops-scorer');

const campaign = {
  subject: 'URGENT: Your account will be suspended',
  sender: 'noreply@fake-bank.com',
  content: 'Click here within 15 minutes or lose access...',
  hasDeadline: true,
  claimsAuthority: true,
  requestsMoney: true,
  hasEmotionalTrigger: true
};

const score = scoreCampaign(campaign);
// Likely returns 76-95 (overwhelming manipulation)
```

---

## Response Actions

### Automated Response (score >= 76)

1. **Block source IP** (if network-based)
2. **Generate Hall of Shame blog post** with PsyOps score
3. **Publish threat intelligence** to STIX feed
4. **Add to Cloudflare firewall** blocklist
5. **Document evidence** (screenshots, transcripts, recordings)

### Alert Response (score 51-75)

1. **Flag for review** by security team
2. **Document indicators** for analysis
3. **Monitor for campaign escalation**
4. **Consider blocking** if additional indicators appear

### Monitor Response (score 26-50)

1. **Log event** for trend analysis
2. **Watch for coordinated activity**
3. **No immediate blocking** (may be false positive)

### Ignore (score 0-25)

1. **Technical attack** with no manipulation indicators
2. **Normal traffic** patterns
3. **No PsyOps concern**

---

## SIEM Integration

### Splunk Query

```spl
index=threat_intelligence psyopsScore>=76
| stats count by src_ip, country, severity_level, psyopsScore
| sort -psyopsScore
```

### Elastic (ECS Format)

```json
{
  "event": {
    "kind": "alert",
    "category": ["intrusion_detection"],
    "type": ["info"],
    "action": "psyops-manipulation-detected"
  },
  "threat": {
    "indicator": {
      "type": "ipv4-addr",
      "ip": "1.2.3.4",
      "confidence": "High"
    },
    "enrichments": [
      {
        "indicator": {
          "type": "custom",
          "description": "PsyOps manipulation score",
          "psyops_score": 85
        }
      }
    ]
  },
  "tags": ["psyops", "social-engineering", "manipulation"]
}
```

### Microsoft Sentinel (KQL)

```kql
ThreatIntelligenceIndicator
| where AdditionalInformation contains "psyopsScore"
| extend PsyOpsScore = toint(parse_json(AdditionalInformation).psyopsScore)
| where PsyOpsScore >= 76
| project TimeGenerated, IndicatorId, ThreatType, PsyOpsScore, Description
| sort by PsyOpsScore desc
```

---

## STIX 2.1 Export

```json
{
  "type": "indicator",
  "spec_version": "2.1",
  "id": "indicator--[uuid]",
  "created": "2025-11-21T00:00:00.000Z",
  "modified": "2025-11-21T00:00:00.000Z",
  "name": "PsyOps Manipulation Campaign - [Threat Name]",
  "description": "Detected psychological manipulation campaign scoring 85/95 on 20-point NCI framework",
  "pattern": "[ipv4-addr:value = '1.2.3.4']",
  "pattern_type": "stix",
  "valid_from": "2025-11-21T00:00:00.000Z",
  "labels": ["malicious-activity", "psyops", "social-engineering"],
  "kill_chain_phases": [
    {
      "kill_chain_name": "mitre-attack",
      "phase_name": "initial-access"
    }
  ],
  "confidence": 85,
  "x_dugganusa_psyops": {
    "score": 85,
    "severity_level": "OVERWHELMING MANIPULATION",
    "indicators": {
      "emotional_manipulation": 5,
      "urgency": 5,
      "authority_overload": 5,
      "missing_information": 5,
      "financial_gain": 5,
      "no_alternatives": 4,
      "framing_manipulation": 5,
      "rapid_behavior_shifts": 5
    },
    "framework": "NCI Engineered Reality Scoring System",
    "epistemic_humility_cap": 95
  }
}
```

---

## False Positive Mitigation

### Common False Positives

1. **Legitimate marketing emails** with urgency (sales, promotions)
   - Mitigation: Check domain reputation, sender history
   - Threshold: Score typically 26-50, not 76+

2. **Genuine password reset emails** claiming authority
   - Mitigation: Verify sender domain matches service
   - Threshold: Low score if no urgency/missing info

3. **News articles** with emotional content
   - Mitigation: Check for journalistic standards (attribution, sources)
   - Threshold: Should score <50 if properly sourced

### Tuning Recommendations

- **Baseline your environment** - Run scoring on known-good traffic
- **Adjust thresholds** - May need 80+ for high-noise environments
- **Whitelist trusted senders** - Reduce false positives from internal comms
- **Monitor score distribution** - Legitimate traffic clusters 0-40

---

## Real-World Examples

### Example 1: IRS Phone Scam (Score: 85/95)

**Scenario:** Caller claims to be IRS Criminal Division demanding immediate payment

**High-Scoring Indicators:**
- Emotional Manipulation (#2): 5/5 - Fear of arrest
- Urgency (#8): 5/5 - "Pay now or police dispatched"
- Authority Overload (#7): 5/5 - Claims federal agent, badge number
- Financial Gain (#10): 5/5 - Demanding $12,450
- Missing Information (#4): 5/5 - No callback number, no case reference
- Suppression of Dissent (#11): 4/5 - "Don't delay or face consequences"
- Logical Fallacies (#15): 5/5 - IRS doesn't call demanding immediate payment

**Total:** 85/100 → Capped at 95%

**Response:** BLOCK, DOCUMENT, PUBLISH WARNING

### Example 2: High-Pressure Sales Ad (Score: 52/95)

**Scenario:** "LAST CHANCE! Only 3 items left!"

**Moderate-Scoring Indicators:**
- Urgency (#8): 5/5 - Countdown timer
- Novelty (#9): 4/5 - "Revolutionary product"
- Cherry-Picked Data (#14): 4/5 - Only positive reviews
- Financial Gain (#10): 5/5 - Selling product
- Emotional Manipulation (#2): 3/5 - FOMO

**Total:** 52/100

**Response:** MONITOR (may be aggressive marketing, not scam)

### Example 3: Academic Paper (Score: 18/95)

**Scenario:** Peer-reviewed research with citations

**Low-Scoring Indicators:**
- Urgency (#8): 1/5 - No time pressure
- Emotional Manipulation (#2): 1/5 - Factual presentation
- Authority Overload (#7): 2/5 - Appropriate citation
- Cherry-Picked Data (#14): 2/5 - Limitations acknowledged
- Missing Information (#4): 1/5 - Full context provided
- Logical Fallacies (#15): 1/5 - Sound reasoning

**Total:** 18/100

**Response:** IGNORE (legitimate content)

---

## Attribution

**Original Framework:** NCI Engineered Reality Scoring System
**Source:** Consumer Finance Review Board (consumerfinancereviewboard.org)
**Adaptation:** DugganUSA LLC - Threat Intelligence Application

**Standing on Shoulders:**
- The Why Files - PsyOps Episode (YouTube)
- Shawn Ryan's Ironclad X PsyOps Series
- DARPA Psychological Operations Research
- Robert Cialdini: "Influence: The Psychology of Persuasion"
- Daniel Kahneman: "Thinking, Fast and Slow"

**Novel Contribution:**
- Democratization (free public framework, no paywall)
- Integration with threat intelligence pipelines
- STIX 2.1 export format
- 95% epistemic humility cap
- Multi-SIEM query templates

---

## Philosophy

**Democratic Sharing:** Hoarding scam detection is morally indefensible. Zero marginal cost for digital goods = zero excuse for gatekeeping.

**Epistemic Humility:** We guarantee a minimum of 5% bullshit exists in any complex system. Claiming 100% certainty is itself a manipulation tactic.

**Pattern Recognition:** Mind control isn't magic. It's predictable psychological levers that can be measured, scored, and defeated through transparency.

---

## References

1. **Framework Documentation:** `/skills/psyops-detection/SKILL.md`
2. **Scoring Implementation:** `/scripts/lib/psyops-scorer.js`
3. **Blog Integration:** `/skills/story-density-analyzer/SKILL.md` (lines 178-289)
4. **Judge Dredd D6:** `/skills/judge-dredd/SKILL.md` (manipulation transparency metric)

---

**Detection Rule ID:** DUGGANUSA-013
**Classification:** PSYOPS-MANIPULATION-CAMPAIGN
**Version:** 1.0.0
**Effective:** November 21, 2025
**Maintained By:** Patrick Duggan, DugganUSA LLC
**License:** CC0-1.0 (Public Domain)

*This detection rule is forensically accurate and based on established psychological research. Every claim is backed by citations. Come at us with facts, not feelings.*
