---
name: job-search
description: Reads JobMatch/cv.md, derives a strict evaluation profile (hard filters, must-haves, anti-patterns), runs a two-phase web search across non-LinkedIn sources, deep-verifies promising postings, and writes the 10 best-fit live jobs to JobMatch/jobs.md with fit scores and reasoning. If fewer than 10 survive strict criteria, runs a constraint-impact analysis and asks the user which filter to relax. Use whenever the user says "find jobs", "search jobs", "/job-search", "run the job search", "match me against open roles", or asks for a fresh job shortlist after the CV is loaded.
---

# Job Search

Goal: produce a tight, ranked shortlist of 10 live jobs that match the person in `JobMatch/cv.md`. Strict on fit. No LinkedIn (unreliable). Every entry must be a real, verified, currently-open posting.

## Hard rules

- **Read `JobMatch/cv.md` first.** If it doesn't exist, stop and tell the user to run `/cv-ingest` — don't guess a profile.
- **No LinkedIn.** Don't search `linkedin.com/jobs` or fetch `linkedin.com` URLs. They render badly and are often stale.
- **No fabricated postings.** Every job in the output must come from a `WebSearch` result that you then verified with `WebFetch`. If a link 404s or the role is closed, drop it. Never invent a company, title, or URL.
- **Confirm criteria before searching.** Show the parsed evaluation profile and let the user tweak it. Searching is expensive — get alignment first.
- **Two-phase filter.** Phase 1 = headline/snippet scan from `WebSearch`. Phase 2 = `WebFetch` on shortlist to verify against full JD. Don't list anything that hasn't passed phase 2.
- **Output is exactly 10 jobs.** If you can't find 10 that pass the rubric, list what you have and say so — don't pad with weak matches.
- **Never silently overwrite `JobMatch/jobs.md`.** If it exists, show last-run date + count and ask: replace, append a fresh dated section, or cancel.

## Steps

### 1. Load and parse the CV into an evaluation profile

Read `JobMatch/cv.md`. Extract into this structured profile (in your head / scratch — not yet written to disk):

```
HARD FILTERS  (a single fail kills the match)
  - seniority_band: [from Targeting → Seniority, expand ±1 level]
      e.g. "PM (IC)" → accept: APM, PM, Senior PM. Reject: Director+, Intern.
  - geography_mode: [from Targeting → Geography]
      e.g. "Global / open" → accept everywhere. "Mumbai hybrid" → only Mumbai or remote-IN.
  - deal_breakers: [verbatim from cv.md]
      e.g. "no crypto, no heavy infra" → reject if JD mentions crypto/web3/blockchain or infra/devops/k8s/SRE as primary scope.

MUST-HAVES  (at least 2 of these in the JD)
  - role_type matches one of Targeting → Role types
  - core PM responsibilities (discovery, roadmap, stakeholder mgmt, metrics ownership)
  - work mode aligns with geography preference

NICE-TO-HAVES  (boosts score, doesn't gate)
  - Domain affinity with parsed experience (healthtech, insurance, consumer apps)
  - "Optimising for" alignment (e.g. international exposure → multi-region, visa-sponsoring, global product)
  - AI / ML in product if user biased toward AI PM
  - 0→1 / new-product language if user wants founding PM

ANTI-PATTERNS  (auto-reject even if other signals are strong)
  - Senior leadership roles requiring 10+ yrs when user has ~2
  - "Sales engineer", "customer success", "implementation specialist" disguised as PM
  - Defence, gambling, crypto, adult, MLM
  - Roles asking for hands-on coding as primary day job
  - Postings older than 60 days where date is visible

YEARS_OF_EXPERIENCE
  - Compute from earliest professional role in cv.md to today.
  - Reject JDs whose floor is >2 years above this number.
```

Today's date is the assistant's `currentDate` — use it for the YoE math and the staleness check.

### 2. Show the profile and confirm

Print the profile in a compact block. Then use `AskUserQuestion`:
- `looks right, search now`
- `tweak one filter` (then walk through which)
- `cancel`

Don't proceed to search until the user confirms.

### 3. Generate search queries

Build 6–10 queries that cover different role types × different sources. Don't just rephrase — vary the source.

**Sources to target** (use site: operators when helpful):
- Company ATS pages: `site:greenhouse.io`, `site:lever.co`, `site:ashbyhq.com`, `site:workable.com`
- Startup boards: Wellfound (formerly AngelList), YC's Work at a Startup (`workatastartup.com`), Otta / Welcome to the Jungle
- Niche PM boards: PMJobsHQ, Product Hunt Jobs, Mind the Product jobs
- Geo-specific (only if profile demands): Naukri, Instahyre (India); Otta, Honeypot (EU)
- AI-specific (when user biases toward AI PM): `aijobs.com`, `aijobslist.com`, generic search for "AI product manager" + ATS sites

**Query templates** (fill in from profile):
- `"{seniority} product manager" {top_domain} site:greenhouse.io`
- `"AI product manager" remote {seniority} site:ashbyhq.com`
- `"founding product manager" {geo_or_remote} site:lever.co`
- `"product manager" {domain} {city_or_"remote"} 2026`
- `site:workatastartup.com "product manager" {seniority}`

Drop queries that don't fit the profile (e.g. don't search "founding PM" if user excluded that role type).

### 4. Phase 1 — collect candidates from WebSearch

Run each query via `WebSearch`. From results, keep entries where the **title + snippet alone** satisfy hard filters. Reject obvious no-gos at this stage:
- Director / VP / Head titles when seniority cap is PM
- Cities outside profile when geo is locked
- Deal-breaker keywords in the title (e.g. "blockchain", "infrastructure")

Collect 20–30 candidates. Deduplicate by URL and by company+title.

### 5. Phase 2 — deep verify with WebFetch

For each shortlisted candidate, `WebFetch` the URL with a focused prompt:
> *"Extract: exact job title, company, location, work mode, seniority/years required, posting date, top 5 responsibilities, top 5 requirements, any disqualifiers (industry, tech stack), application link. If the page is closed/expired/404, say so."*

For every candidate that does not survive, **tag it with exactly one canonical cut-reason** from the list below. This tagging is mandatory — step 7 cannot run without it. If a candidate fails on multiple axes, tag the *most binding* one (the one that would still cut it even if others were relaxed).

```
CUT REASONS (canonical set — pick exactly one)
  seniority_too_high       Title/level above PM-band ceiling (Director, VP, Staff, Principal, Lead, Head, Group, Sr Manager).
  seniority_too_low        Intern, "college grad only", "new grad" when profile demands beyond entry.
  yoe_floor_above_ceiling  JD requires more years than profile.YoE_ceiling. Title may be in-band but the years required are not.
  geography_excluded       JD location is outside the profile's geography setting (e.g. US-only when profile is "Mumbai hybrid").
  no_visa_sponsorship      JD explicitly states no sponsorship AND profile asks for international exposure.
  deal_breaker_industry    Crypto/web3, defence, gambling, adult, MLM, or any industry the user listed.
  deal_breaker_tech        Heavy infra/devops/k8s/SRE/platform-infra as primary scope when user excluded those.
  language_requirement     JD demands fluency in a language the candidate doesn't speak (e.g. French-only).
  domain_misfit            Role scope is genuinely unrelated to anything in the CV and user did not say "open / domain-agnostic".
  closed_or_404            Page expired, deadline past, or returns the company's listing page instead of the role.
  unverifiable             Page didn't render content (Ashby, YC's Work at a Startup, Lever 403s) and we couldn't confirm openness/details.
  stale                    Posting date visible and >60 days old.
```

Keep a structured tally of cuts as you go — at minimum: `{reason: count, examples: [{company, title, url}]}`. You will use this in step 7.

### 6. Score and rank

For each survivor, score on this rubric (max 10):

| Component | Weight | Notes |
|---|---|---|
| Seniority fit | 2 | Exact = 2, ±1 band = 1, mismatch = drop |
| Role-type match | 2 | One of Targeting → Role types = 2, adjacent = 1 |
| Domain affinity | 2 | Strong overlap with parsed experience = 2, neutral = 1, none = 0 |
| Geography fit | 1 | Aligned with profile = 1 |
| "Optimising for" alignment | 2 | Hits the user's stated optimisation = 2 |
| Posting freshness & recruiter signal | 1 | <30d, named hiring manager, real ATS link = 1 |

Tie-break by freshness, then by company quality (named-recognition / well-funded / clearly hiring).

Take the **top 10**. If fewer than 10 survive, **do not write the file yet** — go to step 6.5.

### 6.5 Constraint-impact analysis & adaptive relaxation

Trigger only when survivors after scoring < 10. Goal: surface the constraints that are most binding the pool, propose the smallest defensible relaxation that would unlock the most candidates, and let the user choose.

**a) Build the impact report.** From the cuts tally:
- Sort cut-reasons by count, descending.
- For each top reason, identify *what specific relaxation* would convert those cuts back into candidates. Examples:
  - `yoe_floor_above_ceiling` × 8 → "raise YoE ceiling from 4 → 5 yrs"
  - `seniority_too_high` × 6 (mostly "Senior" titles, none Director+) → "include Senior PM titles even if YoE doesn't quite stretch"
  - `geography_excluded` × 5 (all US-only) → "accept US-only roles even though they don't carry visa sponsorship"
  - `no_visa_sponsorship` × 4 → "drop the international-exposure requirement for this run"
  - `unverifiable` × 12 (Ashby/YC) → "trust phase-1 snippets for Ashby/YC sources, mark as 'snippet-verified' in output"

Drop reasons where the relaxation would violate a hard deal-breaker (e.g. `deal_breaker_industry` is never proposed for relaxation).

**b) Show the report and propose 2–4 ranked relaxations.** Print one line per reason with count + examples + proposed relaxation. Then use `AskUserQuestion` (multi-select) with the top 2–4 relaxation options. Always include a `"don't relax — ship what we have"` option as one choice.

Example phrasing for the question:
> *"Only N matches survived. The biggest constraints right now are: YoE ceiling (cut 8), seniority cap (cut 6), unverifiable Ashby/YC pages (cut 12). Which would you relax?"*

For each option, include:
- the count it would unlock
- a one-line explanation of the trade-off (what could leak in if we relax this)

**c) Re-score with the relaxed profile.** Don't re-search. Pull cuts whose reason matches the user's chosen relaxation back into the candidate pool, run them through the rubric (with the relaxed bound), and re-rank. If a relaxed candidate's content was never verified (e.g. an `unverifiable` Ashby page), score it but **mark it `[snippet-verified]`** in the output rather than `[verified]` — never silently promote.

**d) Loop guard.** Run this relaxation step at most twice per session. After two rounds, ship whatever you have. Never relax a deal-breaker without an explicit second confirmation in chat.

**e) Record the relaxations.** Add a `Relaxations applied` line to the file front-matter in step 7 so the user sees what was loosened.

### 7. Write `JobMatch/jobs.md`

Use this layout:

```markdown
# Found Jobs — {YYYY-MM-DD}

## Profile used for this search
- Seniority: …
- Role types: …
- Domains: …
- Geography / mode: …
- Deal-breakers: …
- Optimising for: …
- YoE computed: …
- Relaxations applied: {none, or list — e.g. "raised YoE ceiling 4 → 5; trusted Ashby/YC snippets"}

## Search queries run
- `…`
- `…`

## Top {N} matches

### 1. {Title} — {Company}    `[Fit: X/10]` `[verified | snippet-verified]`
- **Link**: {url}
- **Source**: {greenhouse / lever / wellfound / etc.}
- **Location**: {city, mode}
- **Posted**: {date if visible, else "date not shown"}
- **Why it fits**:
  - {bullet, anchored in CV ↔ JD overlap}
  - {bullet}
- **Gaps / risks**:
  - {bullet — be honest, e.g. "asks 4 yrs, candidate has ~2"}
- **Score breakdown**: seniority {x/2} · role {x/2} · domain {x/2} · geo {x/1} · optimising {x/2} · freshness {x/1}

(repeat per match, ranked high → low)

## Cut from shortlist
- {Title} — {Company} — {one-line reason cut} — {url}
- (3–5 entries max, only the close-but-no calls)

## Notes
- {anything the user should know: e.g. "couldn't reach 10, only 7 survived phase 2"}
```

### 8. Confirm and exit

Print:
- Path written: `JobMatch/jobs.md`
- N matches written, M cut for honesty
- Top 1–3 by name so the user sees the headline result instantly

Tell the user the next skill (CV-tweak / apply) will read this file. End.

## On re-runs

If `JobMatch/jobs.md` exists:
1. Read its first 20 lines (date + count).
2. Show: *"Existing jobs.md from {date}, {N} matches. Replace, append a new dated section below, or cancel?"*
3. On `append`, add a new `# Found Jobs — {today}` block at the top of the file rather than overwriting.

## Failure modes — handle explicitly

- **CV missing** → stop, point user to `/cv-ingest`.
- **Search returns mostly aggregator noise** (Indeed listicles, "top 10 PM jobs in 2026" blog posts) → reject those, re-query with tighter `site:` operators.
- **Every result is LinkedIn** → reformulate query with `-site:linkedin.com` and prefer ATS-specific operators.
- **Fewer than 10 survive** → run step 6.5 (constraint-impact analysis & adaptive relaxation). Only after the user has either picked relaxations or said "ship what we have" do you write the file.
- **Cuts pile is dominated by `unverifiable`** (Ashby / YC's Work at a Startup / Lever 403s) → propose the "trust phase-1 snippets for these sources" relaxation in step 6.5; in the output, mark those entries `[snippet-verified]` so the user knows to spot-check.
- **A user's "global / open" preference produces too many results** → segment by region (e.g. 4 India, 3 EU, 3 US/remote-global) so the shortlist isn't dominated by one geography.
