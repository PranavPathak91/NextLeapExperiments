---
name: cv-job-matcher
description: Reads JobMatch/cv.md and JobMatch/jobs.md, lets the user pick which roles to analyse, and produces a per-role brief with (a) a calibrated shortlist probability for the current CV, (b) a ranked list of specific CV tweak recommendations with projected probability uplift per tweak, and (c) honest gaps that no tweak can close. Writes one file per role to JobMatch/match/<role-slug>.md. Does NOT rewrite the CV — recommendations only. Use whenever the user says "match my CV to roles", "tweak suggestions", "/cv-job-matcher", "score me on these jobs", or "what should I change on my CV for X".
---

# CV → Job Matcher

Goal: for each role the user picks from `JobMatch/jobs.md`, tell them honestly **how likely they are to be shortlisted as-is**, and what specific changes to their CV would move that probability up the most. Recommendations only — never write the new CV content for them.

## Hard rules

- **Never modify `cv.md`.** This skill is read-only on the CV. The user (or a future tweak skill) does the rewriting.
- **Never fabricate experience.** If a JD asks for X and the CV has no X, the gap goes in "Honest gaps", never in "tweaks." Reframing what's already there is fine; inventing what isn't is not.
- **No generic advice.** Every recommendation must cite (a) a specific CV location (section / role / bullet number) and (b) a specific JD signal it's serving. "Add more metrics" is banned. "QI Spine bullet 1 already says 83%; reorder to position 1 because Wallapop JD's first responsibility is conversion-experiment design" is the bar.
- **Probability is calibrated, not gamed.** Use the bucket rubric below. Don't add 10pp for every minor tweak — uplifts should reflect realistic recruiter behaviour. Stack-rank tweaks by uplift × effort; show effort.
- **Hard floors don't move with tweaks.** If JD requires 5 yrs and CV has 2, no amount of bullet-reordering changes that. Surface it in "Honest gaps" and cap the post-tweak ceiling honestly.
- **Read both inputs first.** If `JobMatch/cv.md` or `JobMatch/jobs.md` is missing, stop and route the user to `/cv-ingest` or `/job-search`.

## Steps

### 1. Load inputs

- Read `JobMatch/cv.md` in full.
- Read `JobMatch/jobs.md` in full. Parse the "Top N matches" section — extract per-role: title, company, link, location, fit score, why-it-fits bullets, gaps/risks bullets.
- Build a quick mental model of the candidate (YoE, top 3 domains, named tools, optimising-for, deal-breakers).

If either file is missing or empty: stop, tell the user which one, point them to the right skill.

### 2. Pick which roles to analyse

Default: top 3 by fit score. But ask the user — they may want a specific role, or all 10.

Use `AskUserQuestion` (multi-select where >2 options):
- `Top 3 by fit` (default)
- `All 10`
- `Pick specific roles` → if chosen, list the 10 with fit scores in a follow-up `AskUserQuestion`

Cap at 5 roles per run unless user explicitly says "all" — analysing all 10 produces a lot of files and the marginal value drops fast.

### 3. For each picked role, build the match brief

Per role, work through the analysis in this order. Don't skip steps — they feed each other.

#### 3a. Re-fetch the JD if needed

If the role's JD details in `jobs.md` are thin (especially `[snippet-verified]` ones), use `WebFetch` on the link with this prompt:
> *"Extract: top 5 must-have requirements, top 5 nice-to-haves, exact YoE floor, named tools/methods, named domains, scope (team size / area), seniority indicators, and 3 phrases the JD repeats most often. If page is JS-rendered or 404, say so."*

For deep-verified roles where `jobs.md` already has the breakdown, skip the re-fetch — use what's there.

#### 3b. Extract the JD's "must-hit signals"

A recruiter scanning a CV in 30 seconds is checking specific signals. Identify them per role:

```
HARD FLOORS (binary — pass/fail)
  - YoE floor
  - Required degree / certification (rare)
  - Work authorisation / location
  - Language requirements

KEYWORD HIT-LIST (recruiter / ATS scan)
  - Tools named in JD (e.g. SQL, Jira, Amplitude, Mixpanel)
  - Methods named in JD (A/B testing, discovery research, OKRs)
  - Domains named (fintech, healthcare, insurance)

NARRATIVE FIT (human reader)
  - Top 3 responsibilities — does the CV's headline + first 2 roles obviously map?
  - Scale signals — does the JD imply users / revenue / team size that CV demonstrates?
  - Seniority cues — leadership, ambiguity, 0→1, scope ownership

DIFFERENTIATORS
  - Unusual JD phrases ("regulatory compliance", "0→1", "self-service") that match CV verbatim
  - Domain crossover the JD will value highly (e.g. UAE compliance for any global health/fintech expansion role)
```

#### 3c. Score the current CV against the signals

Use a 6-component breakdown adding to 100 percentage points. Be calibrated, not generous.

| Component | Weight | What full marks looks like |
|---|---|---|
| Hard floors | 25 | Every binary pass; no work-auth or YoE blocker |
| Keyword coverage | 20 | All JD-named tools/methods present in CV verbatim |
| Domain match | 20 | At least one full-time role in JD's domain (or close adjacency) |
| Narrative fit | 15 | Top 3 CV bullets obviously map to top 3 JD responsibilities |
| Scale & seniority cues | 10 | CV demonstrates user / team / scope at JD's expected level |
| Differentiators | 10 | At least one unusual signal the JD will weight heavily |

Sum → raw score. Map to a calibrated probability bucket:

| Raw score | Probability bucket | Interpretation |
|---|---|---|
| 80+ | 60–80% | Top of pile; CV polish only |
| 65–79 | 40–60% | Strong fit; tweaks meaningfully help |
| 50–64 | 20–40% | Long shot; must reframe to land |
| 35–49 | 8–20% | Weak; consider whether worth applying |
| <35 | <8% | Don't apply unless personal connection |

**Always state the probability as a range, not a single number** ("~35–45%"). False precision is worse than honest ranges. Recruiter behaviour is noisy.

#### 3d. Generate ranked tweak recommendations

Identify the gaps between current score and 100. For each gap, produce ONE recommendation. Categories of recommendation (use these labels in output — they help the user spot what kind of work each is):

- `[Reorder]` — move existing content; zero rewrite cost, often biggest uplift
- `[Reframe]` — same fact, different lens / vocabulary; low cost, medium uplift
- `[Surface]` — content already in CV but buried; lift it into headline or top bullet
- `[Quantify]` — bullet has no number; user knows the number, just hasn't included it
- `[Add keyword]` — tool/method/domain that's true of the user's experience but not literally in the CV
- `[Cut]` — drop or shorten content that crowds out JD-relevant material
- `[Cluster]` — create or reorganise a skill block
- `[Tailor headline]` — rewrite the one-line positioning for this specific role

Per recommendation, output:

```
[Category] {one-line title}
  Where:    {section / role / bullet number in cv.md}
  What:     {specific change — describe it, don't write the new copy}
  Why:      {which JD signal this serves, quoted from the JD if possible}
  Lift:     +{X}pp → {new probability range}
  Effort:   {low / medium / high}
```

Rules for assigning lift:
- **Reorder** of an existing top-3 bullet: +3 to +6pp
- **Reframe** a single bullet: +2 to +5pp
- **Surface** a buried scope/scale signal: +3 to +7pp
- **Quantify** a bullet that's currently vague: +2 to +4pp
- **Add keyword** that's truthfully part of the role: +1 to +3pp per keyword (max +5pp total)
- **Cluster / Tailor headline**: +3 to +8pp depending on JD-CV gap
- **Cut** that creates room for a stronger signal: +0 directly, but unlocks +2-4pp via what fills the space

Cap total tweak uplift at **+25pp**. CV polish doesn't turn a 30% role into a 70% role — recruiters scan for floors first.

Sort recommendations by **lift / effort**. Show top 5–8. Don't list more than that — the user needs a focused punch list, not a wishlist.

#### 3e. Compute the post-tweak probability

Sum the lifts of the recommendations the user could realistically apply (assume they apply the top 5). Add to current probability. Cap at the level the hard floors allow:

- If JD's YoE floor is N years above CV's actual: cap at 50%.
- If domain has zero overlap: cap at 60%.
- If geography fails and no sponsorship: cap at 25% regardless.

State the post-tweak number as a range: *"~45–55% if the top 5 are applied; ceiling of 60% because [hard floor]."*

#### 3f. List honest gaps

Things no tweak fixes for this role. Include:
- YoE shortfall in years
- Missing core experience (e.g. JD requires "led ML pipeline" — CV has no ML)
- Geographic / visa floors
- Required certifications

Frame as decisions, not failures: *"6-9 yrs floor vs ~2 yrs — apply only if the recruiter has a softer band; otherwise this is education-not-application."*

### 4. Write per-role brief to `JobMatch/match/<slug>.md`

Slug format: `{company-lower-kebab}-{role-keyword}.md` (e.g. `kikoff-product-manager.md`, `voize-founding-pm.md`). Don't include date — re-running just overwrites.

Use this layout exactly:

```markdown
# Match: {Role} — {Company}

_Analysed: {YYYY-MM-DD}_

## Job at a glance
- **Link**: {url}
- **Location / mode**: …
- **YoE floor**: …
- **Hard floors**: …
- **Top 3 JD signals (what the recruiter scans for)**:
  1. …
  2. …
  3. …

## Current CV — shortlist probability: ~{X}–{Y}%

**Why this range:**
- ✓ {1-line evidence supporting fit, with CV citation}
- ✓ {…}
- ✗ {1-line evidence against, with JD citation}
- ▲ {ambiguous signal that could go either way once reframed}

**Score breakdown**: hard floors {x/25} · keywords {x/20} · domain {x/20} · narrative {x/15} · scale {x/10} · differentiators {x/10}

## Recommended tweaks  (ranked by lift ÷ effort)

### 1. [Category] {title}    `+Xpp · effort: low`
- **Where**: {section / role / bullet}
- **What**: {specific change — described, not written}
- **Why for this JD**: {signal it serves, JD phrase if possible}

### 2. …

(5–8 recommendations max)

## Post-tweak probability: ~{X}–{Y}%

Cap reasoning: {hard floor that limits the ceiling, if any}

## Honest gaps (no tweak fixes these)
- {gap with one-line framing as a decision}
- …

## Verdict
{One sentence — apply with tweaks / apply only if recruiter contact / pass}.
```

### 5. Write the index summary

After writing all per-role files, append (or rewrite) `JobMatch/match/INDEX.md`:

```markdown
# Match Briefs — {YYYY-MM-DD}

| # | Role | Company | Current % | Post-tweak % | Verdict | Brief |
|---|---|---|---|---|---|---|
| 1 | … | … | ~X | ~Y | apply / pass / contact | [link](kikoff-product-manager.md) |
| 2 | … |
```

Sort by post-tweak probability, descending. This is the user's apply-priority list.

### 6. Confirm and exit

Print:
- Files written: N
- Highest current %: {role} at {X}%
- Highest post-tweak %: {role} at {Y}%
- One sentence: where the user should focus first.

Tell the user the next skill (apply / draft outreach) will read these briefs. End.

## On re-runs

If `JobMatch/match/` already has files for the picked roles:
1. Show: *"Existing briefs for {N} roles, last analysed {date}. Refresh, only update changed roles, or cancel?"*
2. On `refresh`, overwrite. On `partial`, only re-run for roles whose JD was re-verified or whose CV section changed materially.

## Failure modes

- **CV missing** → `/cv-ingest` first.
- **jobs.md missing or empty** → `/job-search` first.
- **JD is `[snippet-verified]` and re-fetch fails** → produce the brief with a `[low-confidence]` banner; recommendations stay generic to category-level signals; tell the user to spot-check before applying.
- **Two roles at same company** (e.g. Razorpay's many PM listings) → produce one brief each; flag the overlap so user doesn't write redundant cover content.
- **User picks all 10** → produce all 10 briefs but warn that file count gets unwieldy and most users only realistically apply to 3–4.
