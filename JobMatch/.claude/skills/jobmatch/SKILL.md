---
name: jobmatch
description: End-to-end JobMatch orchestrator — runs the full pipeline (CV ingest → web job search → per-role CV-tweak recommendations) for a candidate from scratch. Detects which steps have already been done in this JobMatch/ folder and offers to skip, refresh, or replace each. Designed for cohort-style reuse — point it at a fresh JobMatch/ folder and it walks the user from "I have a CV" to "here's a ranked, tweak-suggested apply list" without manual skill chaining. Use whenever the user says "run jobmatch", "start jobmatch", "/jobmatch", "process my CV end-to-end", "do the full pipeline", or kicks off the workflow for someone new.
---

# JobMatch Orchestrator

Goal: take a candidate from zero to an actionable apply-priority list in one invocation. Chain the three existing skills (`cv-ingest`, `job-search`, `cv-job-matcher`), preserve their existing confirmations, and never silently overwrite work the user already did.

## What this orchestrator owns

- **State detection** — figures out which artifacts already exist in `JobMatch/` and adapts the flow.
- **Step gating** — asks the user before running, refreshing, or skipping each step. Never auto-overrides.
- **Hand-off coherence** — after each step, briefly confirms the artifact landed and meets the next step's preconditions before continuing.
- **Final wrap** — summarises what was produced and what the candidate's next concrete action is.

## What this orchestrator does NOT own

- Any of the underlying logic. The sub-skills define their own rules. This file does not duplicate them.
- File rewrites. Each sub-skill writes its own outputs.
- CV editing. Recommendations come from `cv-job-matcher`; this skill never modifies `cv.md`.

## Hard rules

- **Always invoke sub-skills via the `Skill` tool** with these exact names: `cv-ingest`, `job-search`, `cv-job-matcher`. Don't reimplement their steps inline.
- **One confirmation per step.** Use `AskUserQuestion` to get a single decision per sub-skill (run / skip / refresh), not nested questions.
- **Skip-by-default for completed steps** — if `JobMatch/cv.md` already exists and the user just wants jobs, don't force a re-run of cv-ingest.
- **Never proceed past a failed precondition.** If `cv.md` is missing or empty when `job-search` is about to run, stop and route back to `cv-ingest`.
- **Never run all three blindly without user confirmation at the start.** First message of every run should show state and confirm scope.
- **Don't analyse data the sub-skills produce.** Hand off cleanly. The orchestrator is plumbing, not a fourth analyst.

## Steps

### 0. Orient — detect current state

Read the filesystem (use `Bash ls` or equivalent — don't read each file in full):

```
JobMatch/
  cv.md                              ← cv-ingest output
  jobs.md                            ← job-search output
  match/
    INDEX.md                         ← cv-job-matcher output (index)
    *.md                             ← per-role briefs
```

Build a one-line state summary, e.g.:
> *"State: cv.md exists (last modified {date}, ~{N} roles parsed). jobs.md exists ({M} matches, last run {date}). match/ has {K} briefs. Latest INDEX last analysed {date}."*

Or, on a clean slate:
> *"State: empty JobMatch/ folder. Will run all 3 steps: ingest CV → find jobs → match."*

### 1. Confirm scope

Use `AskUserQuestion` once to decide the run shape:

- `Run end-to-end` (recommended for new candidates) — runs all three steps in order, skipping steps whose outputs already exist and are fresh.
- `Refresh everything` — re-runs all three steps even if outputs exist (will prompt before overwriting in each sub-skill).
- `Pick specific steps` → follow up with a multi-select for which of `cv-ingest` / `job-search` / `cv-job-matcher` to run.
- `Just summarise current state` — read existing artifacts, print a one-page summary, exit. No sub-skill invocations.

Use the `Just summarise` option as the auto-response when the user re-runs the orchestrator on a folder where everything is already complete and they didn't say "refresh."

### 2. Step 1 — CV ingest

Decision tree:
- If `cv.md` does NOT exist → **must run**. Invoke `cv-ingest` via the `Skill` tool. After it returns, verify `JobMatch/cv.md` was written; if not, stop and tell the user.
- If `cv.md` exists AND user picked `Refresh everything` → invoke `cv-ingest` (it has its own re-run flow that handles overwrites).
- If `cv.md` exists AND user picked `Run end-to-end` → **skip with a one-line confirmation**: *"Using existing cv.md from {date} — not refreshing."*
- If user picked `Pick specific steps` → run only if cv-ingest is in the picked set.

Precondition for proceeding to step 2: `cv.md` exists and is non-empty. If it's empty after a run, stop and report.

### 3. Step 2 — Job search

Decision tree (same logic as step 2 with `jobs.md`):
- Missing → must run, invoke `job-search`.
- Exists + `Refresh` → run.
- Exists + `Run end-to-end` → skip with note.
- `Pick specific steps` → run if picked.

Precondition: `jobs.md` exists with at least one role under `## Top N matches`. The `job-search` skill is allowed to produce <10 if it relaxed everything and still came up short — that's fine, don't gate on count.

### 4. Step 3 — CV-job match

Same skip/run/refresh logic, applied to `JobMatch/match/INDEX.md`.

When invoking `cv-job-matcher`, do not pre-specify which roles — let the sub-skill's own selection question fire (Top 3 / Top 5 / All 10 / Pick). The orchestrator's job is to invoke, not to decide for the user.

Precondition: `match/INDEX.md` exists. If `cv-job-matcher` errored partway, surface that — don't pretend completion.

### 5. Final wrap

After all selected steps complete, print a single concise summary:

```
JobMatch run complete.
- cv.md            : {created | refreshed | unchanged}  ({N} roles, {M} TODOs)
- jobs.md          : {…}  ({K} matches, top: {company1}, {company2}, {company3})
- match/INDEX.md   : {…}  (top apply: {role at company} — post-tweak {X-Y%})

Next concrete actions for the candidate:
  1. Open match/INDEX.md — that's the apply-priority list.
  2. For each Tier-1 role, apply the tweaks listed in its brief, then apply.
  3. Spot-check the [low-confidence] briefs by opening their links manually.
```

Don't editorialise beyond that. The artifacts speak for themselves.

## Failure modes — handle explicitly

- **No `JobMatch/` folder at all** → create it (`mkdir -p JobMatch`), then start fresh from step 1. Don't error.
- **Sub-skill not registered** (skill name unknown to the harness) → tell the user the skill is missing and where it should live (`JobMatch/.claude/skills/{name}/SKILL.md`). Don't try to inline-implement it.
- **User aborts mid-flow** (says "stop", "cancel") → stop cleanly. Print which steps completed and which didn't, so they can pick up later by re-running.
- **Sub-skill produces empty output** (e.g. `cv-ingest` writes a 0-byte cv.md, or `job-search` produces zero roles even after relaxation) → stop, report the failure honestly, don't invoke downstream steps. Downstream steps depend on real input.
- **Network failure during `job-search`** → the search skill should already handle and tell the user; orchestrator just doesn't proceed to match.

## On re-runs

The orchestrator is idempotent by design — re-running on a populated folder defaults to "summarise current state" unless the user picks `Refresh everything` or `Pick specific steps`. So a cohort member who's already finished can re-run without losing work.

If the user wants to start a *new candidate* in the same repo without overwriting the previous one, suggest they either:
- Move the existing `JobMatch/cv.md`, `jobs.md`, and `match/` to `JobMatch/archive/{candidate-name}/` first, OR
- Run the orchestrator from a different working directory with its own `JobMatch/` folder.

Don't auto-archive — that's destructive and ambiguous. Surface the choice and let the user act.

## Cohort-reuse notes (for the operator running this for multiple people)

- Each candidate should have their own `JobMatch/` working directory — either a separate clone, a separate worktree, or `JobMatch/people/<name>/` with the orchestrator pointed at that subfolder. The skill itself doesn't enforce isolation; that's a directory-management decision.
- Skills are loaded from `.claude/skills/` relative to the working directory. So `cd JobMatch && claude` will auto-load all four skills. If you run from elsewhere, the skills won't be discovered.
- The orchestrator's outputs (cv.md, jobs.md, match/) are deterministic-shaped — they're safe inputs to a future `apply` skill, an outreach-drafter, or any other downstream tool you build.
