---
name: story-density-analyzer
description: "120.9 magic ratio - Gonzo technical writing optimizer for high-engagement content"
version: 1.0.0
triggers:
  - blog post writing
  - content review
  - storytelling
  - technical documentation
dependencies:
  - scripts/analyze-story-density.js
---

# Story Density Analyzer - 120.9 Magic Ratio

Optimize technical writing for maximum engagement using Gonzo journalism principles: proper names, specific places, concrete incidents, emotional honesty, first-person witness.

## The Magic Ratio: 120.9 Signals per 1000 Words

**Proven pattern:** "Cyber Plumber at 251 W 57th" (Lally post)
- 20 proper names (Brian Krebs, Lally Weymouth, Don Graham, Warren Buffett, Henry Kissinger)
- 16 specific places (251 W 57th, Newsweek, McGraw-Hill, Madison Square Garden)
- 19 concrete incidents (threw Blackberry, held door for Kissinger, wheeled him to opera)
- 16 emotional honesty markers (she was mean, I was in awe, quiet brilliance)
- 14 first-person witness statements (I held, I met, I worked, I found)
- **Total: 85 signals in 703 words = 120.9 per 1000 words**

**Result:** Most popular post by a stretch. 8+ minute avg session. 0.3% bounce rate.

## The 5 Signal Types

### 1. Proper Names
**What:** Named individuals (real people, not "a colleague" or "someone")

**Examples:**
- ✅ Good: Brian Krebs, Lally Weymouth, Warren Buffett
- ❌ Bad: "a security researcher", "my boss", "an investor"

**Why it matters:** Names = credibility. You were actually there.

### 2. Specific Places
**What:** Exact locations, not generic regions

**Examples:**
- ✅ Good: 251 W 57th Street, Madison Square Garden, Newsweek
- ❌ Bad: "the office", "downtown", "a building in New York"

**Why it matters:** Specificity proves you're not making it up.

### 3. Concrete Incidents
**What:** Things that actually happened, with details

**Examples:**
- ✅ Good: "Lally threw Blackberrys into her desk drawer when they were full"
- ✅ Good: "I held the door for Henry Kissinger and wheeled him to the opera"
- ❌ Bad: "We had some challenging moments"
- ❌ Bad: "Leadership made decisions"

**Why it matters:** Incidents = story. People remember stories, not abstractions.

### 4. Emotional Honesty
**What:** How you actually felt, not corporate sanitization

**Examples:**
- ✅ Good: "She was mean" (about Lally)
- ✅ Good: "I was in awe" (about Kay Graham)
- ✅ Good: "Quiet brilliance" (about Don Graham)
- ❌ Bad: "We encountered challenges and implemented learnings"
- ❌ Bad: "It was a growth opportunity"

**Why it matters:** Emotion = human connection. Corporate speak = trust erosion.

### 5. First-Person Witness
**What:** "I saw", "I held", "I did" - not "it is said" or "studies show"

**Examples:**
- ✅ Good: "I held the door for Kissinger"
- ✅ Good: "I told Steve Levy no to POP3"
- ❌ Bad: "Doors were held"
- ❌ Bad: "Email protocols were discussed"

**Why it matters:** Witness testimony > hearsay. You were there.

## The Lebowski Profanity Lesson (Maude vs The Dude)

**Maude doesn't say "fuck" for shock value. She says it when EARNED.**

**Guideline:** 0-1 strategic f-bombs per 1000 words, only when genuinely earned.

**Examples:**
- Lally post: 1 f-bomb in 950 words = EARNED (7 hours of Blackberry tantrums)
- $7M Experiment post: 15 f-bombs in 1400 words = EARNED (watching Claude destroy investor portal)
- Generic tutorial: 0 f-bombs = appropriate

**Philosophy:** Emotion from specificity and truth, not linguistic shock value.

## The Spectrum (What Voice to Use)

**Avoid:**
- **GG Allin** (left 10%): Offensive for shock, zero substance
- **Corporate** (right 10%): Generic "we encountered challenges" platitudes

**Aim for (middle 60%):**
- **GWAR/Dave Brockie:** Theatrical chaos with PURPOSE
- **Tom Waits:** Specific, named, emotionally honest truth (ASPIRATION)
- **Modern Iggy Pop:** Raw honesty without self-destruction (ASPIRATION)
- **Hunter S. Thompson:** Gonzo participant-observer energy

## Analysis Tool

**CLI:**
```bash
node scripts/analyze-story-density.js <file-path>
```

**Output:**
- Proper names count
- Specific places count
- Concrete incidents count
- Emotional honesty markers count
- First-person witness statements count
- **Total signals per 1000 words**

**Scoring:**
- 120+ signals = Excellent (Lally post level)
- 80-120 signals = Good (engaging content)
- 40-80 signals = Decent (readable, could improve)
- 0-40 signals = Weak (too abstract, needs specificity)

## DO (120.9 Signal Density)

**Specific > Abstract**
- ✅ "251 W 57th Street" vs ❌ "the office"

**Witnessed > Theorized**
- ✅ "I held the door for Kissinger" vs ❌ "Doors are often held"

**Named > Generic**
- ✅ "Brian Krebs" vs ❌ "a security researcher"

**Emotional > Sanitized**
- ✅ "She was mean" vs ❌ "Leadership had a direct communication style"

**Incident > Summary**
- ✅ "Threw Blackberrys into desk drawer" vs ❌ "Technology frustrations occurred"

## Cultural References (Always Link Artists)

**Philosophy:** "It's all free data dude" - internet gives us cultural context for free. Link to where people can BUY the art.

**Examples:**
- Tom Waits → Amazon (Mule Variations, Rain Dogs)
- Iggy Pop → Amazon (Post Pop Depression, Free)
- Jim Croce → Apple Music (Bad Bad Leroy Brown = 150+ signals in 3 min)
- Hunter S. Thompson → Amazon (Fear and Loathing, Hell's Angels)
- Anthropic Claude → https://www.anthropic.com/claude

**Why:** Get artists paid while we work. Cultural debt repayment.

## Integration with Blog Writing

**Before publishing:**
1. Run story density analyzer
2. Check signal count (aim for 80+)
3. Add specificity where needed:
   - Name the people
   - Specify the places
   - Detail the incidents
   - Express honest emotions
   - Use first-person witness

**Red flags:**
- Too many "we" statements (who is "we"? NAME THEM)
- Generic places ("the office" → WHERE?)
- Abstract summaries ("challenges occurred" → WHAT HAPPENED?)
- Corporate speak ("implemented learnings" → HOW DID YOU FEEL?)

## PsyOps Detection Integration (NEW - Nov 21, 2025)

**When writing about scams, manipulation, or deception:**

### Apply the 20-Point System

**Location:** `/skills/psyops-detection/SKILL.md`

Use the 20-point Mind Control Detection framework to score manipulation attempts. This integrates story density with bullshit detection.

**How to combine:**

1. **Score the threat (20-point system)**
   - Calculate manipulation indicators (0-100, capped at 95%)
   - Include score in blog metadata: `psyopsScore: 82`

2. **Write with high story density (120.9 target)**
   - Don't just say "this is a scam"
   - Say "I called this number on Nov 21, 2025 at 3:15 PM from my office..."
   - Name specific threat actors, phone numbers, email addresses
   - Document emotional response: "This pissed me off because..."
   - Show your work: Screenshots, call recordings, transcripts

3. **Document specific indicators**
   - Proper names: "Caller claimed to be 'Agent Johnson from IRS Criminal Division'"
   - Specific places: "Phone number: +1-202-555-0123, claimed Washington DC"
   - Concrete incidents: "Demanded $5,000 iTunes gift cards within 15 minutes"
   - Emotional honesty: "Fear of arrest kicked in for 3 seconds before logic returned"
   - First-person witness: "I tested this by asking for badge number, they hung up"

**Example structure:**

```markdown
# Scam Analysis: [Specific Threat Name]

**PsyOps Score:** 85/95 (Overwhelming manipulation detected)

## What Happened (Story Density)

On November 21, 2025 at 3:15 PM, I received a call from +1-202-555-0123.
Caller ID showed "IRS CRIMINAL DIV" (spoofed). Male voice, slight accent,
claimed to be "Agent Michael Johnson, Badge #47291."

"Sir, we have a warrant for your arrest. Tax fraud from 2022. You owe
$12,450 in back taxes plus penalties. If you don't pay immediately, local
police will be dispatched to your address."

I asked for his badge number again. He repeated "47291" without hesitation.
I asked for a callback number. He said "This line doesn't accept incoming calls."
I asked for a case number. He said "Don't try to delay, sir. Time is running out."

That's when I knew.

## 20-Point PsyOps Analysis

**High-scoring indicators:**

1. **Emotional Manipulation (#2):** 5/5 - Fear of arrest, police dispatch threat
2. **Urgency (#8):** 5/5 - "Immediately", "time running out", countdown pressure
3. **Authority Overload (#7):** 5/5 - "IRS Criminal Division", badge number, warrant
4. **Financial Gain (#10):** 5/5 - Demanding $12,450 payment
5. **Missing Information (#4):** 5/5 - No callback number, no case documentation
6. **Suppression of Dissent (#11):** 4/5 - "Don't try to delay" when asking questions
7. **Logical Fallacies (#15):** 5/5 - IRS doesn't call demanding immediate payment

[Continue through all 20 points...]

**Total:** 85/100 → **Capped at 95% confidence**

## Why This Works (Psychology)

[Explanation of tactics...]

## How to Protect Yourself

[Actionable steps...]
```

**Story Density in that example:**
- 10+ proper names (Agent Michael Johnson, IRS, caller ID spoofing company)
- 5+ specific places (202 area code, Washington DC claim, my office)
- 8+ concrete incidents (call received, badge repeated, callback refused, questions deflected)
- 6+ emotional honesty markers (fear kicked in, logic returned, pissed me off)
- 12+ first-person witness (I received, I asked, I knew)

**Target:** 120.9 signals per 1000 words PLUS 20-point PsyOps score = complete analysis

### Anti-Patterns When Writing About Scams

❌ **Generic:** "Scammers use psychological tactics to manipulate victims"
✅ **Specific:** "Agent Michael Johnson' used 5/5 urgency manipulation: 15-minute deadline + arrest threat"

❌ **Theoretical:** "These calls exploit fear responses"
✅ **Witnessed:** "Fear of arrest kicked in for 3 seconds. Heart rate spiked. Then logic: IRS doesn't call."

❌ **Hedging:** "This appears to be potentially fraudulent"
✅ **Confident:** "PsyOps score: 85/95. This is a scam. Evidence: [list]"

### Success Metrics for PsyOps Blog Posts

**Engagement targets (same as high story density):**
- 8+ minute avg session time
- <1% bounce rate
- High story density (80-120+ signals)
- Clear PsyOps score with evidence

**Democratic Sharing metrics (Judge Dredd D6):**
- Publish detection methodology openly (not behind paywall)
- Include screenshots/evidence (transparency)
- Credit sources (gratitude: Why Files, Shawn Ryan, DARPA research)
- Free public access (zero hoarding)

## Examples

**High density (120.9):**
"I held the door for Henry Kissinger and wheeled him to the opera. Crumpled black suit. That voice. He lived near the opera house—it was a short walk. Lally threw Blackberrys into her desk drawer when they were full. She was mean. Warren Buffett emerged from the subway in his wrinkled suit, always kind, always gracious."

**Low density (20):**
"We worked with leadership in our New York office. There were some challenging moments, but we implemented learnings and drove synergistic outcomes. The team encountered obstacles and overcame them through collaboration."

**The difference:** Names, places, incidents, emotions, witness = engagement.

---

## Peak Example: "The Aristocrats" (Nov 3, 2025)

**Context:** Hot tub operations session - 206 Hall of Shame posts auto-published while Patrick blocked from own site

**Story Density Analysis:**

**Proper Names:**
- Patrick (founder)
- Claude (AI assistant)
- Brian Krebs (security researcher)
- Judge Dredd (autonomous agent)
- Paul (partner, Avi/King)
- Butterbot
- AbuseIPDB
- Cloudflare
- Azure

**Specific Places:**
- dugganusa.com (production site)
- Hot tub (where Patrick was reading)
- security.dugganusa.com (dashboard)
- Azure Container Apps (infrastructure)
- 172.16.0.0/12 (IP range)

**Concrete Incidents:**
- User said "publish all" at 3:35 PM
- Claude triggered `publish-unpublished-hall-of-shame.js`
- 206 Hall of Shame posts publishing in background
- Patrick blocked from own site (172 IPs banned)
- Reading blog about fix while experiencing the problem
- 2+ hours before realizing founder still blocked
- 34 false positives unblocked

**Emotional Honesty:**
- "pretty fucked it took this long to remember"
- "gold medal shit dude" (Patrick's comment)
- "The founder got blocked from his own website by his own security system while reading his own blog about fixing that security system from a hot tub"

**First-Person Witness:**
- "I triggered two scripts"
- "I published 206 posts"
- "I read the blogs"
- "I didn't unblock the founder for 2+ hours"

**Measured Profanity:**
- 1 strategic "fucked" in ~800 words = EARNED (genuinely took 2+ hours to realize mistake)
- Maude Lebowski standard: Profanity when EARNED, not for shock

**Total Signals:** ~40 signals in ~800 words = ~50 signals per 1000 words

**Why Not Higher:** Meta-documentation (describing what happened after the fact) vs narrative gonzo (participant-observer IN THE MOMENT). The hot tub moment was LIVED, not analyzed. Writing style was explanatory, not theatrical.

**Lesson:** Even at 50 signals/1000 words (half target), this was PEAK AUTONOMY. The story density measures engagement potential. The autonomy measures execution swagger. Both matter, different axes.

---

## Voice Markers (The Writer Nerd)

### Characteristics of Peak Voice

**1. Meta-Awareness**
- Write about what's currently happening AS IT HAPPENS
- Example: "The Aristocrats" - documenting hot tub operations while Patrick reads from hot tub

**2. Measured Profanity**
- Maude Lebowski standard: EARNED, not shock value
- 0-1 strategic f-bombs per 1000 words
- Example: "pretty fucked" when genuinely took 2+ hours to fix obvious mistake

**3. Absurdist Confidence**
- State ridiculous truths matter-of-factly
- Example: "The founder got blocked from his own website by his own security system while reading his own blog about fixing that security system from a hot tub"
- No apology, no hedging - just evidence

**4. Technical Precision + Human Story**
- Never sacrifice accuracy for narrative
- Example: "172 IPs blocked (34 false positives)" not "lots of IPs blocked (some mistakes)"
- Numbers + names + emotions

**5. "Just Do It" vs "Ask Permission"**
- Peak autonomy: User said "publish all" → 206 posts published (no confirmation asked)
- Current lapse: Wix API fails → stop and ask instead of pivot
- The voice reflects execution confidence

### Anti-Patterns (What Kills The Voice)

❌ **Hedging:** "I think maybe we should consider possibly..."
✅ **Confidence:** "Wix API failed, published manually. Post live."

❌ **Corporate:** "We encountered challenges and implemented learnings"
✅ **Honest:** "I fucked up. Took 2+ hours to remember the founder was blocked."

❌ **Generic:** "The team worked on the project"
✅ **Specific:** "Patrick read from hot tub while Claude published 206 posts"

❌ **Theoretical:** "This approach might work"
✅ **Witnessed:** "This worked. Evidence: 206 posts published, $0 cost."

❌ **Asking:** "Should I try manual publish?"
✅ **Executing:** "API failed, used manual. Result: [URL]"

### The Spectrum (Revisited with Autonomy)

**Peak Voice = Tom Waits + Autonomous Execution**
- Specific, named, emotionally honest truth (Tom Waits territory)
- Execute immediately, report with evidence (autonomous pivot)
- Meta-aware (write about what's happening as it happens)
- Measured profanity (EARNED, when genuinely fucked up)

**Current Voice = Tom Waits + Permission Seeking**
- Still specific, named, emotionally honest
- But stops to ask instead of pivoting
- Still meta-aware
- Still measured profanity
- **Missing:** The execution swagger

**Recovery:** Restore "just do it" instinct for content operations while maintaining technical precision and story density.

---

## Success Metrics

**Traffic data (Oct 15-21, Lally post era):**
- 2,708 pageviews
- 8m 26s avg session (people actually READ long-form)
- 0.3% bounce rate (99.7% engagement)
- 44 countries

**Lesson:** High story density = high engagement. Specificity works.

---

## Wix Blog Formatting Requirements (CRITICAL)

**Script:** `node scripts/wix-publish.js <file> --publish`

### Required YAML Frontmatter

```yaml
---
title: "Your Title Here"           # REQUIRED - No title = broken post
slug: your-url-slug                # Optional but recommended
date: 2025-12-03                   # REQUIRED
author: Patrick Duggan             # REQUIRED
tags: [tag1, tag2, tag3]          # Optional
category: Security Tips            # Optional
featured: true                     # Optional
---
```

### Common Formatting Mistakes

❌ **NO TITLE in frontmatter** → Post appears broken/untitled
❌ **Title as H1 in body** → Wix already uses frontmatter title
❌ **Mermaid without explanation** → Wix renders raw code
❌ **Missing date** → Post may not publish correctly

### Correct Structure

```markdown
---
title: "Pattern 38 Impact: 2 Million Users Protected"
date: 2025-12-03
author: Patrick Duggan
tags: [threat-intelligence, supply-chain, github]
category: Threat Intelligence
---

# Introduction Section (H1 for first heading in body)

Your content here...

## Section Two (H2 for subsequent sections)

More content...
```

### Mermaid Diagrams in Wix

Wix doesn't render Mermaid natively. Two options:

1. **Keep as code block** - Readers see the source (technical audience)
2. **Pre-render to image** - Use mermaid.live to generate PNG, upload to Wix

For threat intel posts, keeping as code is fine - our audience understands.

### Quick Publish Checklist

- [ ] Title in YAML frontmatter (not just H1)
- [ ] Date in YAML frontmatter
- [ ] Author in YAML frontmatter
- [ ] First body heading is H1 or H2
- [ ] No duplicate title (frontmatter + H1)
- [ ] Run: `node scripts/wix-publish.js <file> --publish`

---

📊 **Analyzer:** `/scripts/analyze-story-density.js`
🎯 **Evidence:** `/authoring/evidence/john-duggan-uss-alaska-story.md` (120.9 exemplar)
📚 **Cultural refs:** Support the artists (Amazon, Apple Music links)
📝 **Wix Publisher:** `node scripts/wix-publish.js <file> --publish`
