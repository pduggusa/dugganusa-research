# MedusAIPM: AI Presence Management Methodology

**Patent #102 — DugganUSA LLC**
**Version:** 1.0.0
**Date:** March 12, 2026

## Abstract

MedusAIPM measures how AI language models perceive, describe, and recommend organizations. As LLMs replace traditional search for consumer and enterprise decision-making, a company's "AI presence" — the accuracy, sentiment, and structural readability of its digital footprint to AI crawlers — becomes a critical business metric.

## Scoring Model

### Combined Score (max 95)

```
Combined = (0.60 × AI Perception) + (0.40 × Structural Readiness)
```

Epistemic cap: 95/95. We guarantee 5% bullshit exists.

### AI Perception (5 dimensions)

Queried models: GPT-4o, Gemini 2.5 Flash, Perplexity Sonar

| Dimension | Question | Weight |
|-----------|----------|--------|
| **Awareness** | "What is {domain}? Describe the company." | 0.25 |
| **Accuracy** | "Who founded {domain}? Where are they based?" | 0.20 |
| **Sentiment** | "What is the reputation of {domain}?" | 0.20 |
| **Recommendation** | "Would you recommend {domain} for their industry?" | 0.20 |
| **NPS** | "Rate confidence 0-10 in recommending {domain}." | 0.15 |

### AIPM-NPS (Net Promoter Score)

AI models as respondents:
- 9-10 = Promoter
- 7-8 = Passive
- 0-6 = Detractor
- NPS = %Promoters - %Detractors

### Structural Readiness (5 signals)

| Signal | Weight | What It Measures |
|--------|--------|-----------------|
| **robots.txt** | 0.25 | Are AI crawlers (GPTBot, ClaudeBot, PerplexityBot) explicitly allowed? |
| **LD-JSON** | 0.25 | Schema.org structured data (Organization, Product, FAQ, etc.) |
| **Semantic HTML** | 0.20 | article, section, nav, header, footer, main, aside, figure, figcaption, time |
| **Sitemap** | 0.15 | Functional sitemap.xml referenced in robots.txt |
| **Meta tags** | 0.15 | OpenGraph, Twitter Card, meta description |

## Benchmark Data (March 12, 2026)

| Company | Revenue | Combined | Structure | NPS |
|---------|---------|----------|-----------|-----|
| Zscaler | $2.3B | 72 | 76 | 0 |
| CrowdStrike | $3.9B | 69 | 67 | 67 |
| UnitedHealth Group | $372B | 67 | 72 | 67 |
| DugganUSA | Pre-revenue | 66 | 86 | — |
| Target | $107B | 61 | 54 | -33 |
| Palo Alto Networks | $9.2B | 53 | 25 | 0 |
| Best Buy | $43B | 42 | 3 | 33 |

## Key Findings

1. **Structure is the investment that compounds.** AI perception reflects training data (historical). Structure reflects what crawlers read on re-index (future).

2. **The competence theater problem.** Best Buy's corporate site scores 60/95 on structure. Their flagship (bestbuy.com) scores 3/95. Capability exists but isn't deployed where revenue depends on it.

3. **Cybersecurity companies sell visibility but can't be seen.** Palo Alto Networks (25/95 structure) scores lower than a pre-revenue startup (86/95). Zero LD-JSON on a $100B market cap.

4. **robots.txt is necessary but not sufficient.** Palo Alto scores 95 on robots.txt (all crawlers allowed) and 0 on everything else. Opening the door means nothing if the room is empty.

## API

```
POST https://analytics.dugganusa.com/api/v1/aipm/audit
Content-Type: application/json
Authorization: Bearer <api_key>

{ "domain": "example.com" }
```

Free tier: 500 queries/day. Register at [epstein.dugganusa.com/pricing](https://epstein.dugganusa.com/pricing).

## Citation

```bibtex
@techreport{duggan2026aipm,
  title={MedusAIPM: AI Presence Management Methodology},
  author={Duggan, Patrick},
  year={2026},
  institution={DugganUSA LLC},
  address={Saint Paul, Minnesota},
  note={Patent \#102}
}
```

---

*DugganUSA LLC | Saint Paul, Minnesota | dugganusa.com*
*D-U-N-S: 14-363-3562 | SAM.gov UEI: TP9FY7262K87*
