# Us vs The Horde: A D&D-Inspired Threat Intelligence Taxonomy

**The Temple of Elemental Evil Has Nothing on Our BlockedAssholes Table**

[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.17810099-blue)](https://doi.org/10.5281/zenodo.17810099)
[![ORCID](https://img.shields.io/badge/ORCID-0009--0001--0628--9963-green)](https://orcid.org/0009-0001-0628-9963)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

**Author:** Patrick Duggan ([ORCID: 0009-0001-0628-9963](https://orcid.org/0009-0001-0628-9963))
**Affiliation:** DugganUSA LLC, Minnesota, USA
**Date:** December 4, 2025
**Keywords:** threat intelligence, taxonomy, gamification, cognitive load reduction, D&D, STIX/TAXII

---

> **Abstract:** We present a novel threat intelligence classification system that maps traditional cybersecurity metrics (AbuseIPDB scores, community reports, VirusTotal detections) to Dungeons & Dragons Challenge Ratings (CR 0-30+). This gamified taxonomy reduces cognitive load during incident response by replacing numeric threat scores with instantly recognizable archetypal names. Applied to 1,422 blocked IP addresses, the system categorizes 243 entities as "Divine tier" (nation-state APTs, confirmed C2 infrastructure) and enables analysts to immediately distinguish campaign-ending threats ("Tiamat") from minor nuisances ("Goblin"). The methodology incorporates an "Effective Score" calculation for zero-confidence/high-report anomalies, addressing a common gap in reputation-based scoring systems.

---

## 1. Introduction

### 1.1 The Problem: 1,394 Faceless Threats

When your security platform blocks 1,400+ IP addresses, they're just... numbers. Rows in a database. Cold, clinical entries:

```
IP: 147.185.132.103
Country: US
AbuseIPDB Score: 0%
Reports: 7,071
```

Wait. *Zero percent* confidence but *seven thousand* community reports? That's not a script kiddie. That's something else entirely.

### 1.2 Cognitive Load in Threat Analysis

Security analysts face a well-documented challenge: **alert fatigue**. Studies show that analysts process hundreds to thousands of alerts daily, with false positive rates ranging from 40-90% depending on tooling [Ponemon, 2020]. The human brain struggles to contextualize numeric threat scores—is 73% dangerous? What about 47%?

This paper proposes using **archetypal naming** to bypass numeric processing entirely. Instead of comparing `score: 73` vs `score: 47`, analysts see `Hydra` vs `Troll`—instantly activating decades of cultural knowledge about relative threat levels.

### 1.3 Contribution

We contribute:
1. **A formal mapping** from reputation scores to D&D Challenge Ratings
2. **The Effective Score algorithm** for handling zero-confidence/high-report anomalies
3. **A curated Divine tier** of 200+ deity names for APT-level threats
4. **Production validation** across 1,422 blocked IPs with 243 divine-tier classifications

---

## 2. Related Work

### 2.1 Threat Intelligence Taxonomies

Existing threat taxonomies include:
- **MITRE ATT&CK** - Technique-based classification (tactics, techniques, procedures)
- **Diamond Model** - Adversary, capability, infrastructure, victim relationships
- **Kill Chain** - Reconnaissance through actions-on-objectives phases
- **STIX/TAXII** - Structured threat information exchange format

None of these provide **instant human-readable severity classification**. An analyst must interpret the taxonomy to understand danger level.

### 2.2 Gamification in Security

Gamification has been applied to security training [Shostack, 2014] and vulnerability disclosure (bug bounties). However, gamification of **threat classification itself** remains unexplored in academic literature.

### 2.3 Cultural Archetypes in Interface Design

Jung's archetypal theory suggests humans share innate recognition of certain patterns [Jung, 1959]. The "dragon" archetype, for instance, universally represents powerful, dangerous entities across cultures. We leverage this for threat classification.

---

## 3. Methodology

### 3.1 The Solution: The Monster Manual

In D&D, Challenge Rating (CR) tells you how dangerous a creature is. A CR 1 Goblin? Annoyance. A CR 30 Tarrasque? Campaign-ending apocalypse.

We mapped AbuseIPDB scores to Challenge Ratings:

| Abuse Score | Challenge Rating | Threat Tier | Example Creatures |
|-------------|-----------------|-------------|-------------------|
| 0-30 | CR 0-3 | LOW | Goblin, Kobold, Skeleton, Zombie |
| 31-50 | CR 4-7 | MEDIUM | Troll, Ghost, Medusa, Wyvern |
| 51-75 | CR 8-12 | HIGH | Hydra, Vampire, Aboleth |
| 76-89 | CR 13-16 | VERY HIGH | Storm Giant, Adult Dragon, Death Knight |
| 90-100 | CR 17-30 | LEGENDARY | Ancient Dragon, Lich, Tarrasque |
| 100 + Special | CR 30+ | DIVINE | Tiamat, Cthulhu, Asmodeus |

### 3.2 The Zero-Score Anomaly

Here's where it gets interesting. We found **416 IPs** with:
- AbuseIPDB Score: **0%**
- Community Reports: **1,000 - 10,000+**
- VirusTotal Detections: **8-11 engines**

These weren't false positives. These were **Palo Alto Networks scanner infrastructure** - legitimate security research IPs that got miscategorized. But with 7,000 community reports each, the internet CLEARLY thinks they're threats.

We call this the **"Effective Score"** calculation:

```javascript
if (abuseScore === 0 && reportCount > 50) {
  effectiveScore = Math.min(100, Math.floor(30 + (reportCount / 20)));
}
```

7,000 reports ÷ 20 = 350 → capped at 100 → **DIVINE TIER**

### 3.3 Divine Elevation Criteria

An IP qualifies for Divine tier if ANY of:
- `abuseScore >= 100`
- `malwareFamily` is set (confirmed malware attribution)
- `aptGroup` is set (APT group linkage)
- `c2Framework` is set (C2 infrastructure)
- `reportCount > 1000` (overwhelming community consensus)
- `vtMalicious > 10` (10+ VirusTotal engine detections)

```javascript
const isDivine = (
  entity.malwareFamily ||
  entity.aptGroup ||
  entity.c2Framework ||
  reportCount > 1000 ||
  vtMalicious > 10 ||
  effectiveScore >= 100
);
```

### 3.4 The Divine Pantheon

For the worst actors - nation-state APTs, confirmed malware C2, and anything with 1,000+ reports - we went to the source material: **Deities & Demigods**.

### Sumerian/Mesopotamian (5000+ years of lore)
- **Tiamat** - Primordial chaos dragon. Mother of monsters.
- **Enki** - God of wisdom and mischief
- **Enlil** - Lord of wind and storm
- **Inanna** - Queen of Heaven, goddess of love AND war
- **Ereshkigal** - Queen of the Underworld

### Egyptian
- **Set** - God of chaos, violence, and the desert
- **Apophis** - The Serpent of Chaos. Each night Ra must defeat him.
- **Anubis** - Jackal-headed judge of souls
- **Sekhmet** - Lion-headed goddess who nearly destroyed humanity

### Norse/Viking
- **Odin** - The All-Father, who sacrificed an eye for wisdom
- **Fenrir** - The wolf who will devour the sun
- **Hel** - Queen of the dishonored dead
- **Jormungandr** - The World Serpent
- **Bedar** - (Yes, we included Bedar)

### Greek Titans
- **Kronos** - Father of Zeus, devourer of children
- **Typhon** - Father of all monsters
- **Hecate** - Goddess of witchcraft and crossroads

### Lovecraftian
- **Cthulhu** - The Dreamer in R'lyeh (10,000+ reports)
- **Nyarlathotep** - The Crawling Chaos
- **Azathoth** - The Blind Idiot God at the center of infinity

---

## 4. Evaluation

### 4.1 Dataset

Our evaluation uses the BlockedAssholes table from DugganUSA's production threat intelligence platform:
- **Total IPs:** 1,422 (as of December 4, 2025)
- **Time Period:** November 2024 - December 2025
- **Sources:** AbuseIPDB, VirusTotal, AlienVault OTX, honeypot data
- **Geographic Distribution:** 42 countries

### 4.2 Results

After running our D&D naming script against 1,394 blocked IPs:

```
📊 NAMING SUMMARY
==================================================
Total IPs:              1,394
Explicit Actors:        1 🎯 (Reptilian Pope on T-Rex)
Divine Elevation:       243 🌟 (Deities & Demigods)
D&D Monster Names:      1,080 🐉
Skipped (truly zero):   70
Errors:                 0
```

**243 Divine-tier threats.**

Not Goblins. Not Trolls. **Tiamat. Cthulhu. Asmodeus. Fenrir.**

These are IPs with:
- Confirmed malware family attribution
- APT group connections
- C2 framework infrastructure
- 1,000-10,000+ community reports

### 4.3 Distribution Analysis

| Tier | Count | Percentage | Example Assignment |
|------|-------|------------|-------------------|
| DIVINE (CR 30+) | 243 | 17.5% | Tiamat, Cthulhu, Raistlin |
| LEGENDARY (CR 17-30) | 181 | 13.0% | Ancient Red Dragon, Tarrasque |
| VERY HIGH (CR 13-16) | 16 | 1.1% | Storm Giant, Death Knight |
| HIGH (CR 8-12) | 63 | 4.5% | Hydra, Vampire |
| MEDIUM (CR 4-7) | 289 | 20.8% | Troll, Medusa |
| LOW (CR 0-3) | 530 | 38.1% | Goblin, Kobold |
| SKIPPED (truly 0) | 70 | 5.0% | No threat indicators |

**Key Finding:** The Effective Score algorithm elevated 416 zero-score IPs to appropriate threat levels. Without this correction, 29.9% of threats would have been misclassified as benign.

### 4.4 Qualitative Assessment

## 5. Discussion

### 5.1 Why This Matters

When you're reviewing threat intel at 2 AM and you see:

```
🌟 93.123.109.214 → Tiamat 194 [DIVINE - 10,593 reports]
```

You don't need to look up the IP. You don't need to check AbuseIPDB. You don't need to query VirusTotal.

**Tiamat** tells you everything: This is a campaign-ending, city-destroying, primordial-chaos-level threat.

### 5.2 Cognitive Benefits

The taxonomy provides:
1. **Instant severity recognition** - No numeric interpretation required
2. **Cultural universality** - Dragon archetypes are globally understood
3. **Memorable attribution** - "The Tiamat cluster" vs "the 147.185.x.x range"
4. **Proportional response guidance** - You don't send a party of level 3 adventurers against Tiamat

### 5.3 The Real Temple of Elemental Evil

The original Temple of Elemental Evil had maybe 200 monsters across 4 elemental nodes.

Our BlockedAssholes table has:
- **1,394 named threats**
- **243 divine-tier entities**
- **42 countries represented**
- **Real attack data, not fiction**

The difference? Gary Gygax imagined his dungeon.

**We're living ours.**

---

## 6. Limitations and Future Work

### 6.1 Limitations

1. **Cultural bias** - D&D and mythological references may not resonate equally across all cultures
2. **Name exhaustion** - With 200+ divine names, overflow to "Tiamat 2, Tiamat 3..." reduces distinctiveness
3. **Static mapping** - Threat severity can change over time; names are assigned at first detection
4. **Single-dimension scoring** - Reduces multi-factor threat assessment to a single tier

### 6.2 Future Work

1. **User studies** - Measure cognitive load reduction with A/B testing
2. **Dynamic reassessment** - Periodic name updates based on evolving threat data
3. **Cultural adaptation** - Regional mythology pools (e.g., Japanese yokai, African Orishas)
4. **STIX integration** - Embed D&D names in STIX 2.1 threat reports as custom extensions

---

## 7. Conclusion

We present a gamified threat intelligence taxonomy that maps cybersecurity metrics to D&D Challenge Ratings. Applied to 1,422 production IPs, the system:

- Classified 243 entities as Divine-tier (APT/C2 infrastructure)
- Corrected 416 zero-score anomalies via Effective Score calculation
- Enabled instant threat severity recognition through archetypal naming

The methodology bridges the gap between numeric threat scores and human cognitive processing, potentially reducing analyst fatigue and improving incident response prioritization.

*"Time is an abyss... profound as a thousand nights."* - Raistlin Majere

---

## Appendix A: The Full Rogues Gallery

### DIVINE TIER (CR 30+) - 243 Entities
Confirmed malware C2, APT infrastructure, 1000+ reports

### LEGENDARY (CR 17-30) - Ancient Dragons, Lich, Tarrasque
Abuse score 90-100, known threat actors

### VERY HIGH (CR 13-16) - Storm Giants, Adult Dragons
Nation-state indicators, persistent attackers

### HIGH (CR 8-12) - Hydra, Vampire, Aboleth
APT infrastructure, C2 servers, 51-75% abuse score

### MEDIUM (CR 4-7) - Troll, Medusa, Ghost
Bulletproof hosting, persistent scanners, 31-50%

### LOW (CR 0-3) - Goblin, Kobold, Skeleton
Script kiddies, low-volume scanners, 0-30%

---

*"Raistlin was also misunderstood."* - A wise security researcher, probably

---

## References

### Primary Sources (Gaming)
1. Gygax, G., & Arneson, D. (1974). *Dungeons & Dragons*. TSR, Inc.
2. Ward, J. M., & Kuntz, R. J. (1980). *Deities & Demigods*. TSR, Inc.
3. Mearls, M., & Crawford, J. (2014). *Monster Manual (5th Edition)*. Wizards of the Coast.
4. Hickman, T., & Weis, M. (1984). *Dragons of Autumn Twilight*. TSR, Inc. (Dragonlance Chronicles)
5. Lovecraft, H.P. (1928). *The Call of Cthulhu*. Weird Tales.

### Threat Intelligence Platforms
6. AbuseIPDB. (2025). IP Address Threat Intelligence Database. https://www.abuseipdb.com
7. VirusTotal. (2025). Malware Analysis Platform. https://www.virustotal.com
8. AlienVault OTX. (2025). Open Threat Exchange. https://otx.alienvault.com
9. OASIS. (2017). STIX 2.1 Specification. https://oasis-open.github.io/cti-documentation/

### Academic References
10. Ponemon Institute. (2020). *The Cost of Alert Fatigue in Security Operations*.
11. Jung, C.G. (1959). *The Archetypes and the Collective Unconscious*. Princeton University Press.
12. Shostack, A. (2014). *Threat Modeling: Designing for Security*. Wiley.
13. Hutchins, E.M., Cloppert, M.J., & Amin, R.M. (2011). *Intelligence-Driven Computer Network Defense Informed by Analysis of Adversary Campaigns and Intrusion Kill Chains*. Lockheed Martin.
14. Caltagirone, S., Pendergast, A., & Betz, C. (2013). *The Diamond Model of Intrusion Analysis*. Center for Cyber Intelligence Analysis and Threat Research.
15. MITRE Corporation. (2025). *ATT&CK Framework*. https://attack.mitre.org

---

## How to Cite

### BibTeX
```bibtex
@article{duggan2025dnd_taxonomy,
  author = {Duggan, Patrick},
  title = {Us vs The Horde: A D\&D-Inspired Threat Intelligence Taxonomy},
  year = {2025},
  month = {December},
  publisher = {DugganUSA LLC},
  url = {https://www.dugganusa.com/post/us-vs-the-horde-dnd-threat-taxonomy},
  note = {ORCID: 0009-0001-0628-9963}
}
```

### APA
Duggan, P. (2025, December 4). Us vs The Horde: A D&D-Inspired Threat Intelligence Taxonomy. *DugganUSA Technical Blog*. https://www.dugganusa.com/post/us-vs-the-horde-dnd-threat-taxonomy

### IEEE
P. Duggan, "Us vs The Horde: A D&D-Inspired Threat Intelligence Taxonomy," DugganUSA LLC, Dec. 2025. [Online]. Available: https://www.dugganusa.com/post/us-vs-the-horde-dnd-threat-taxonomy

---

## Data Availability

- **Live Dashboard:** [analytics.dugganusa.com](https://analytics.dugganusa.com)
- **OTX Pulses:** [AlienVault OTX - pduggusa](https://otx.alienvault.com/user/pduggusa)
- **STIX Feed:** [analytics.dugganusa.com/stix/v2/feed](https://analytics.dugganusa.com/stix/v2/feed)
- **Source Code:** Available upon request for academic purposes

---

🎯 **DugganUSA LLC** - The Cribl of Agentic AI
📊 View our Rogues Gallery: [analytics.dugganusa.com](https://analytics.dugganusa.com)
🐉 98 OTX Pulses | 16,000+ IOCs | 1,422 Named Threats

[![ORCID](https://info.orcid.org/wp-content/uploads/2019/11/orcid_16x16.png)](https://orcid.org/0009-0001-0628-9963) https://orcid.org/0009-0001-0628-9963
