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

## The Problem: 1,394 Faceless Threats

When your security platform blocks 1,400+ IP addresses, they're just... numbers. Rows in a database. Cold, clinical entries:

```
IP: 147.185.132.103
Country: US
AbuseIPDB Score: 0%
Reports: 7,071
```

Wait. *Zero percent* confidence but *seven thousand* community reports? That's not a script kiddie. That's something else entirely.

## The Solution: The Monster Manual

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

## But What About The Zeros?

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

## The Divine Pantheon

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

## The Final Tally

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

## Why This Matters

When you're reviewing threat intel at 2 AM and you see:

```
🌟 93.123.109.214 → Tiamat 194 [DIVINE - 10,593 reports]
```

You don't need to look up the IP. You don't need to check AbuseIPDB. You don't need to query VirusTotal.

**Tiamat** tells you everything: This is a campaign-ending, city-destroying, primordial-chaos-level threat.

## The Real Temple of Elemental Evil

The original Temple of Elemental Evil had maybe 200 monsters across 4 elemental nodes.

Our BlockedAssholes table has:
- **1,394 named threats**
- **243 divine-tier entities**
- **42 countries represented**
- **Real attack data, not fiction**

The difference? Gary Gygax imagined his dungeon.

**We're living ours.**

---

## Appendix: The Full Rogues Gallery

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

1. Gygax, G., & Arneson, D. (1974). *Dungeons & Dragons*. TSR, Inc.
2. Ward, J. M., & Kuntz, R. J. (1980). *Deities & Demigods*. TSR, Inc.
3. Mearls, M., & Crawford, J. (2014). *Monster Manual (5th Edition)*. Wizards of the Coast.
4. AbuseIPDB. (2025). IP Address Threat Intelligence Database. https://www.abuseipdb.com
5. VirusTotal. (2025). Malware Analysis Platform. https://www.virustotal.com
6. OASIS. (2017). STIX 2.1 Specification. https://oasis-open.github.io/cti-documentation/
7. Hickman, T., & Weis, M. (1984). *Dragons of Autumn Twilight*. TSR, Inc. (Dragonlance Chronicles)
8. Lovecraft, H.P. (1928). *The Call of Cthulhu*. Weird Tales.

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
