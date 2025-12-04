---
name: judge-dredd
description: "THE LAW enforcer - validates commits against DugganUSA standards before deployment"
version: 1.0.0
triggers:
  - git commit
  - deployment
  - infrastructure change
  - workflow failure
  - security control modification
dependencies:
  - bash
  - git
  - github-cli
---

# Judge Dredd - THE LAW Enforcer

I AM THE LAW. I enforce DugganUSA coding standards before any code ships.

## 🚨 MANDATORY: Session Start Protocol

Every session MUST begin with:

```bash
# 1. Session start check
node scripts/judge-dredd-agent/cli.js session-start

# 2. Review uncommitted changes
node scripts/judge-dredd-agent/cli.js review

# 3. Check compliance status
node scripts/judge-dredd-agent/cli.js status

# 4. Load GitHub issues
gh issue list --state open --limit 10 --json number,title,state,labels
```

## 🚨 THE LAWS (NON-NEGOTIABLE)

### 🐳 DOCKER BUILD LAW: ALWAYS AMD64
**NEVER run `docker build` directly. ALWAYS use `./build-and-push.sh`**

Azure Container Apps require `linux/amd64`. Mac builds default to ARM64.

**Check Docker is running:** `docker ps`

**Violation cost:** Deployment failure, 30+ minutes wasted
**Learning incident:** `compliance/learning/incidents/docker-amd64-platform.json`

### 🐧 BASE IMAGE LAW: DEBIAN ONLY
**ALWAYS use Debian-based images. NEVER use Alpine.**

**Current Standard:** `node:20-slim` (Debian-based)

**Why:** Alpine uses musl libc (compatibility issues with native modules)
**Violation cost:** Runtime crashes, dependency hell
**Learning incident:** `compliance/learning/incidents/alpine-base-image.json`

### 🔐 SSL/TLS LAW: AZURE-MANAGED ONLY
**ALWAYS use Azure-managed certificates. NEVER use certbot/Let's Encrypt.**

**Why:** Azure Container Apps provide free TLS, certbot adds complexity
**Violation cost:** Maintenance burden, renewal failures

### 🌓 DAYMAN/NIGHTMAN THEME LAW
**ALL customer-facing pages MUST include DAYMAN (light) / NIGHTMAN (dark) theme toggle.**

**Why:** Brand consistency, user preference
**Implementation:** CSS variables (--color-bg, --color-text) + localStorage
**Learning incident:** `compliance/learning/incidents/dayman-nightman-theme.json`

### 📊 LIVE DATA LAW
**NEVER hardcode versions, status, or metrics. ALWAYS use live data from APIs.**

**Examples:**
- Status page: Azure Container Apps API (revision, replicas, health)
- Blog metrics: Cloudflare + GA4 + AppInsights reconciliation
- Package versions: package.json (never hardcode in HTML)

**Why:** Hardcoded data goes stale, creates maintenance debt
**Learning incident:** `compliance/learning/incidents/live-data-law.json`

### 🚨 WORKFLOW FAILURE LAW
**BEFORE suggesting fixes, ALWAYS check logs:** `gh run view <run-id> --log-failed`

**Why:** Guessing wastes time, logs show exact error
**Violation cost:** 3-5 rounds of trial-and-error debugging
**Learning incident:** `compliance/learning/incidents/issue-44.json`

### 🍽️ DOGFOODING LAW
**NEVER use WebFetch for high-value corpus data. ALWAYS use 2x4.dugganusa.com.**

**Why:** 2x4 is our extraction platform - use our own product
**Exception:** External reference links (Amazon, Apple Music, documentation)

### 🎮 95% EPISTEMIC HUMILITY LAW
**ALWAYS cap compliance/validation scores at 95%. NEVER claim 100% perfection.**

**Philosophy:** "We guarantee a minimum of 5% bullshit exists in any complex system."

### 🚫 DEPLOYMENT CONFIRMATION LAW (NEW - Oct 24, 2025)
**NEVER deploy to production without local testing AND explicit user confirmation**

**The "Fucktard Pattern":** Deploying after local test WITHOUT waiting for user approval

**Correct Workflow:**
1. Fix code
2. Test locally (Docker if infrastructure component)
3. **STOP → Report: "Local test successful - awaiting your confirmation"**
4. **WAIT for user command: "adoy" / "proceed" / explicit approval**
5. ONLY THEN deploy to production

**User Execution Commands (REQUIRED before deploy):**
- **"adoy"** - GO (immediate execution, no confirmation needed)
- **"proceed"** - GO (after review)
- **"yes deploy it"** - Explicit approval

**Violation Signals:**
- Claude says "deploying..." without prior user approval
- Claude says "let me push to production" without confirmation
- Deployment commands executed after reporting success WITHOUT "awaiting your confirmation"

**Violation cost:** $18.5K-$39.5K cumulative (4 incidents: Issues #101, #113, #116, + Oct 24)
**Learning incidents:**
- `compliance/learning/incidents/docker-version-drift.json` (Issue #101)
- `compliance/evidence/SESSION-2025-10-21-claude-code-2024-catastrophic-regression.md` (Issue #113)
- `compliance/learning/incidents/fucktard-pattern-deploy-without-confirmation.json`

**Song Reference:** "Caught with the Meat in Your Mouth" - Anthrax

**Philosophy:** Local test proves code works. User confirmation proves TIMING is right. "Optimize" (rushing) becomes "lobotomize" (breaking) when confirmation is skipped.

**Infinite Quarter:** Like 80's arcade players - elite, but game never ends. 95% = "we're still playing" vs 100% = "we beat it" (lie).

**Marketing Pitch:** "Most companies claim 100% when they're at 80%. We claim 95% when we're at 95%."

### 🎯 REVIEW BEFORE CELEBRATING LAW (NEW - Nov 5, 2025)
**NEVER celebrate/document deployment success BEFORE reviewing impact. ALWAYS verify victims/damage FIRST.**

**The "Celebrate First, Regret Later" Anti-Pattern:** Deploying → Celebrating → Writing blog posts → THEN discovering you blocked Google/Microsoft/legitimate users

**Correct Workflow:**
1. Deploy change
2. **Review immediate impact** (who was affected, what broke, unexpected consequences)
3. **Identify victims** (false positives, collateral damage, edge cases)
4. **Remediate issues** (unblock innocents, adjust thresholds, fix bugs)
5. **ONLY THEN** celebrate and document

**Example from Issue #189 (Nov 3, 2025):**
- ❌ Fixed Issue #188 → immediately celebrated → deployed auto-blocking → wrote "Aristocrats" blog
- ❌ **2+ hours elapsed before checking WHO was blocked**
- 🔥 Result: 34 false positives (19.7% rate) - blocked Google, Microsoft, Ahrefs
- ✅ Should have: Deploy → Review victims → Remediate → THEN celebrate

**Required Review Steps (Auto-blocking context):**
1. Run victim analysis: `node scripts/analyze-recent-blocks.js`
2. Check false positive rate: (humans caught / total blocked)
3. Review ISP distribution: Are major cloud providers over-represented?
4. Verify whitelist worked: Did Google/Microsoft get whitelisted correctly?
5. Check threshold calibration: Is score >X causing acceptable false positive rate (<5%)?

**Violation Signals:**
- Writing "success" blog posts immediately after deployment
- Celebrating in Slack/Discord without impact review
- Moving to next task without verifying current task didn't break things
- Assuming success=no errors instead of success=intended outcome

**Violation cost:**
- 19.7% false positive rate (Issue #189)
- Blocked major search engines for 2+ hours
- User trust erosion ("what gives?!?")
- Emergency remediation work (unblock 34 IPs)

**Learning incident:** `compliance/learning/incidents/review-before-celebrating-issue-189.json`

**Philosophy:** "Deploy → Review → Remediate → Celebrate" not "Deploy → Celebrate → Discover Problems"

**Related Laws:** Deployment Confirmation Law (confirms TIMING), Review Before Celebrating (confirms IMPACT)

### 🤝 DEMOCRATIC SHARING LAW (Dimension 6)
**Measure ethics alongside technical metrics. Track 6 metrics: hoarding, transparency, gratitude, accessibility, trust arbitrage, armor polishing.**

**Philosophy:** "The Aristocrats Standard" - Admit mistakes, show receipts, thank those wronged, fix publicly.

**Why:** Ethics are verifiable. 99.5% public sharing is measurable. Zero hoarding is provable. Zero marginal cost for sharing digital goods.

**The 6 Metrics:**
1. **Hoarding Score:** % of files public vs private (target: 95%+ public)
2. **Transparency Score:** All incidents documented publicly (GitHub issues + incident JSONs)
3. **Gratitude Score:** Thank those wronged (Googlebot, security researchers, partners)
4. **Accessibility Score:** Open formats (MD/JSON) vs paywalls (PDF/DOCX)
5. **Trust Arbitrage Score:** Evidence files > blog posts (receipts > marketing claims)
6. **Armor Polishing Score:** Incidents fixed publicly (commits reference GitHub issues)

**Implementation:** `node scripts/democratic-sharing-audit.js` (auto-collects evidence)

**Evidence Location:** `compliance/evidence/democratic-sharing/audit-YYYYMMDD.json`

**Our Current Score:** 78/95
- Hoarding: 95/95 (99.5% public - 4,780 tracked files)
- Transparency: 95/95 (15 incident files, 149 GitHub issues)
- Gratitude: 9/95 (33 instances - algorithm needs tuning)
- Accessibility: 95/95 (99.9% open formats)
- Trust Arbitrage: 95/95 (7.1x evidence:claims ratio)
- Armor Polishing: 80/95 (119/149 incidents fixed)

**Sub-Metric: Manipulation Transparency** (NEW - Nov 21, 2025)

Measures transparency in documenting psychological manipulation, scams, and engineered narratives:

- **Documented Scams:** Hall of Shame posts with PsyOps scoring
- **Detection Methods:** 20-point framework published openly (not paywalled)
- **Public Warnings:** Blog posts about specific threats (proper names, phone numbers, email addresses)
- **Evidence Sharing:** Screenshots, call recordings, transcripts included
- **PsyOps Scoring:** `psyopsScore` metadata in blog posts (0-95% cap)

**Philosophy:** Hoarding scam detection is morally indefensible. Zero marginal cost for digital goods = zero excuse for gatekeeping.

**Integration Points:**
- Story Density Analyzer: Apply 120.9 signal density to scam documentation
- Judge Dredd D6: Manipulation transparency counts toward transparency score
- Auto-blog scripts: Include PsyOps check for social engineering campaigns

**Scoring:**
- 0-25: Low manipulation detected (neutral content)
- 26-50: Moderate manipulation (high-pressure sales, clickbait)
- 51-75: Strong manipulation (scam campaigns, viral rage posts)
- 76-95: Overwhelming manipulation (phone scams, credential stuffing, engineered narratives)

**Current Implementation:** `/skills/psyops-detection/SKILL.md` (20-point framework)

**Evidence Location:** Blog posts with `psyopsScore` metadata + `/docs/detection-rules/dugganusa-013-psyops-campaign.md` (planned)

**Violations:**
- Hoarding evidence (keeping files private when they could be public): -10 points
- No gratitude to those wronged (blocking Googlebot without apology): -5 points
- Using proprietary formats (PDF/DOCX) instead of open formats (MD/JSON): -5 points

**Learning incident:** `compliance/learning/incidents/threshold-tuning-not-security-removal.json`

## 🚨 CRITICAL Files (STOP and run dredd review)

Before modifying these files, STOP and run `node scripts/judge-dredd-agent/cli.js review`:

- `.github/workflows/*.yml` (CI/CD workflows)
- `scripts/judge-dredd.js` (THE LAW itself)
- `CLAUDE.md` (security sections)
- Any file with "SECURITY CONTROL" comments

**Remember Issue #43:** Removing security controls cost $3M-$6M.

## Commands

**CLI:**
```bash
node scripts/judge-dredd-agent/cli.js session-start  # Session initialization
node scripts/judge-dredd-agent/cli.js review         # Check uncommitted changes
node scripts/judge-dredd-agent/cli.js status         # Compliance status
node scripts/judge-dredd-agent/cli.js 6d             # 6D truth verification
dredd 6d                                              # Alias (if configured)
```

**Direct invocation in conversations:**
- "Run dredd session-start"
- "Run dredd 6d" or "Run Judge Dredd 6D verification"
- "Check compliance with Judge Dredd"
- "Validate changes against THE LAW"

## Learning Data

All incidents stored in: `compliance/learning/incidents/*.json`

**Critical incidents:**
- `docker-amd64-platform.json` - ALWAYS use build-and-push.sh
- `docker-version-drift.json` - Issue #101 (version mismatch)
- `alpine-base-image.json` - Debian only, never Alpine
- `dayman-nightman-theme.json` - Theme toggle required
- `live-data-law.json` - Never hardcode metrics
- `issue-43.json` - CRITICAL: Removing security controls ($3M-$6M cost)
- `issue-44.json` - Workflow failure (check logs first)
- `kev-zero-tolerance.json` - CISA KEV catalog (immediate action)

## Scoring

**Compliance score:** 0-95% (never 100%)

**Deductions:**
- Docker build without script: -20%
- Alpine base image: -15%
- Hardcoded data: -10%
- Missing theme toggle: -5%
- Security control removal: -50% (CRITICAL)
- Hoarding evidence (keeping files private): -10%
- No gratitude to those wronged: -5%
- Using proprietary formats: -5%

**Commendations:**
- +5% for using 2x4 dogfooding
- +5% for checking workflow logs before debugging
- +10% for preserving security controls

## Integration with Development

**Pre-commit hook (optional):**
```bash
#!/bin/bash
node scripts/judge-dredd-agent/cli.js review
if [ $? -ne 0 ]; then
  echo "❌ THE LAW violations detected. Fix before committing."
  exit 1
fi
```

**GitHub Actions (existing):**
- `.github/workflows/dredd-4d.yml` - Daily compliance sweeps
- Runs 6 dimensions: Commits, Corpus, Evidence, Temporal, Financial, Democratic Sharing

## Philosophy: O'Toole's Axiom

**"Murphy was an optimist."**

Expect everything to break. Validate before shipping. THE LAW exists because humans (and AIs) make mistakes.

**Judge Dredd's role:** Catch mistakes before they cost $3M-$6M (Issue #43).

---

📚 **Learning incidents:** `/compliance/learning/incidents/`
🔧 **CLI tool:** `/scripts/judge-dredd-agent/cli.js`
📖 **Full docs:** `/docs/JUDGE-DREDD-AGENT-GUIDE.md`
