# Architecture & Data Flow — Daily Market Research Agent

**Owner:** Pranav · **Last updated:** 2026-05-10

This doc has two jobs:

1. Show how the system is wired — every byte's path from input to output.
2. Make the *design rules* explicit, so the same approach can be re-applied to the next complex system.

The PRD ([market-research-prd.md](market-research-prd.md)) is the *what*. This is the *how* and the *why this shape*.

---

## 1. What we built, in one paragraph

A local-first market-research agent that runs entirely on Claude Code primitives. A user defines a watchlist (`watchlist.json`); a daily run spawns one parallel subagent per company; each subagent does web research and writes a fixed-template Markdown brief; a final pass synthesises a cross-cutting summary. A local web dashboard reads the same Markdown files. No backend, no DB, no third-party API keys, no auth.

The novelty is not what it does — it's the *substrate*: skills, subagents, hooks, schedules, and the filesystem, composed into a real workflow.

---

## 2. System map

```
┌──────────────────────────────────────────────────────────────────────┐
│                         USER (Pranav, in CC)                          │
│   types /research, /watchlist, /dashboard — or schedule fires it      │
└──────────────┬─────────────────────────────────────┬──────────────────┘
               │                                     │
               ▼                                     ▼
   ┌───────────────────────┐             ┌────────────────────────┐
   │  setup-watchlist      │             │  research              │
   │  SKILL                │             │  SKILL                 │
   │  (config: AskUserQ.)  │             │  (execution)           │
   └───────────┬───────────┘             └───────────┬────────────┘
               │ writes                              │ reads
               ▼                                     ▼
        ┌──────────────────────────────────────────────────┐
        │              watchlist.json                       │
        │  { context, companies[], dimensions[], delivery } │
        │      ── single source of truth for config ──      │
        └──────────────────────────────────────────────────┘
                                     │
                                     │ fans out, one Agent call per company
                                     ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐        ┌──────────┐
   │ subagent │  │ subagent │  │ subagent │  ...   │ subagent │
   │ OpenAI   │  │ Anthropic│  │ Google   │        │ Cohere   │
   │ (parallel)│ │ (parallel)│ │ (parallel)│       │ (parallel)│
   └────┬─────┘  └────┬─────┘  └────┬─────┘        └────┬─────┘
        │   tools: WebSearch, WebFetch, Write              │
        │   reads: yesterday's brief, briefs/_seen.json     │
        ▼   writes: briefs/YYYY-MM-DD/{Company}.md          ▼
   ┌──────────────────────────────────────────────────────────┐
   │                briefs/YYYY-MM-DD/                         │
   │  OpenAI.md  Anthropic.md  Google.md  ...  _summary.md    │
   │           ── filesystem is the database ──                │
   └──────────────────────────────────────────────────────────┘
                                     │
                       ┌─────────────┼──────────────┐
                       ▼             ▼              ▼
              SessionStart      /dashboard      /research-diff
              hook prints       Vite renders    reads YDA + today
              _summary.md       briefs/         flags net-new
              on CC open        in browser
```

---

## 3. The three loops

The system is three independent control loops, each owning a different time horizon. Keeping them decoupled is what makes the design tractable.

### Loop A — Configure (human-in-loop, minutes)

```
user → setup-watchlist skill → AskUserQuestion ×N → watchlist.json
```

Structured Q&A only — no free-text editing. The skill is the *only* writer to `watchlist.json`. Re-running edits in place; never blind-overwrite. Fast path: every step has a default, accepting them all completes in <60s.

### Loop B — Research (machine-driven, ~5 min/day)

```
schedule (08:00) ─┐
                  ├─→ research skill → preflight (read watchlist, _seen, yesterday)
user /research ─┘                          │
                                           ▼
                              spawn N subagents in ONE message (parallel)
                                           │
                                           ▼  each:
                              WebSearch → WebFetch → write brief.md → return JSON
                                           │
                                           ▼
                              parent merges _seen.json, writes _summary.md, prints to chat
```

The whole point of this design is the parallel fan-out. Sequential is wrong — it would take 8× longer and the demo loses its punch.

### Loop C — Read (human-in-loop, seconds)

```
CC open → SessionStart hook → cat briefs/{today}/_summary.md     (passive surface)
user /dashboard → vite dev server → reads briefs/ → localhost:5173 (active surface)
user opens briefs/YYYY-MM-DD/Company.md directly                  (raw surface)
```

Three surfaces, all reading the same Markdown. The dashboard is the *designed* surface; the file tree is the *honest* surface; the SessionStart hook is the *pulse* — proof that the agent ran without you.

---

## 4. State & storage contract

Everything is on disk, in formats a human can grep. No DB, no cache layer.

| Path | Owner | Lifetime | Shape |
|---|---|---|---|
| `watchlist.json` | setup-watchlist skill | persists | config (companies, dimensions, delivery) |
| `briefs/YYYY-MM-DD/{Company}.md` | research subagent | persists | per-company daily brief, fixed template |
| `briefs/YYYY-MM-DD/_summary.md` | research skill (parent) | persists | cross-cutting roll-up |
| `briefs/_seen.json` | research skill (parent) | rolling 30 days | dedupe memory across runs |
| `.claude/skills/*/SKILL.md` | author | versioned in git | executable spec for each skill |
| `.claude/settings.local.json` | author | versioned in git | hooks, permissions |
| `web/` (M4+) | author | versioned in git | Vite + React dashboard |

**Rules of the contract:**

- **Briefs are immutable once written.** Re-running asks before overwriting. This is what makes day-over-day diffs trustworthy.
- **Every brief follows the same template per depth.** No improvisation. A stable template is what makes diffing trivial — freelancing breaks it.
- **Every concrete claim has a source URL inline.** No URL → cut the claim. This rule lives in the skill prompt, not the parent's head.
- **`_seen.json` is append-only within a day**, pruned weekly. Subagents don't write to it directly — the parent merges their returned JSON.

---

## 5. How information actually flows (one daily run, traced byte-by-byte)

1. **Trigger.** Either (a) a scheduled task fires `/research` at 08:00, or (b) Pranav types `/research` in CC.
2. **Skill activation.** CC matches the description in `.claude/skills/research/SKILL.md` and loads it as the active skill.
3. **Preflight (parent, sequential).** Parent agent reads `watchlist.json`, validates it, computes `TODAY`, checks `briefs/{TODAY}/` for existing files, reads `briefs/_seen.json` and yesterday's per-company briefs.
4. **Fan-out (parent, one message).** Parent issues N `Agent` tool calls *in a single message*. Each prompt is built from the template in [SKILL.md](.claude/skills/research/SKILL.md) — fully self-contained: company info, focus areas, lens, dimensions, depth, yesterday's brief inlined, output template, output path, source rules.
5. **Subagent work (parallel, independent).** Each subagent calls WebSearch → WebFetch → drafts → writes to `briefs/{TODAY}/{Company}.md` → returns a small JSON to the parent (`new_items`, `failures`, `file_path`). Subagents have no memory of the parent conversation; the prompt is the entire context.
6. **Merge (parent, sequential).** Parent collects all JSON returns, merges `new_items` into `_seen.json`, prunes >30 day entries.
7. **Synthesis (parent).** Parent reads all briefs it just wrote, composes `briefs/{TODAY}/_summary.md` using the fixed summary template.
8. **Surface (parent).** Parent prints a 5–10 line chat summary: date, coverage, NEW count, top 3, file paths, any failures.
9. **(Async) Read.** Pranav opens CC tomorrow → SessionStart hook prints `_summary.md`. Or runs `/dashboard` → Vite serves `briefs/` at localhost:5173. Or just opens the Markdown file.

That's it. Every step is a Claude Code primitive doing exactly one thing.

---

## 6. The design rules we followed

These are the rules that produced this shape. They are reusable — apply them to the next complex system.

### Rule 1 — Pick the smallest substrate that works

We chose: filesystem, Markdown, JSON, slash commands, subagents, hooks, schedules. We rejected: database, queue, hosted backend, auth layer, third-party APIs.

**Why:** every layer you skip is a layer you don't have to debug, secure, deploy, or pay for. The product only exists if it's *demoable in 60 minutes from an empty folder.* The substrate enforces that.

**How to apply:** before adding infra, ask "what does this *enable* that I cannot already do with a file?" If the answer is "nothing essential for this milestone," skip it.

### Rule 2 — Files are the contract

State lives in files: `watchlist.json` (config), `briefs/*.md` (output), `_seen.json` (memory). Every skill is itself a file (`SKILL.md`).

**Why:** files are inspectable, diffable, version-controllable, and *survive every layer of tooling*. A user can grep, edit, copy, share, restore from git, or render to a website — all without our help. A DB row can do none of that.

**How to apply:** when you need persistence, the default is "a file, in a directory, in a known shape." Reach for a DB only when the access pattern (concurrent writes, indexed queries, transactions) genuinely needs it. For one-user, append-mostly, read-occasionally workloads — files win.

### Rule 3 — Decouple the loops

Three loops (configure, research, read) write and read independently. The configure loop never executes research. The research loop never edits config. The read loop is read-only.

**Why:** coupled loops mean changes in one cascade through all of them. Decoupled loops mean each can evolve independently and fail independently. The dashboard could be rewritten in Svelte tomorrow with zero impact on the agent.

**How to apply:** identify the natural time horizons in your system (interactive, recurring, on-demand). Each gets its own component. Communication between them is *only through the shared file contract.*

### Rule 4 — Parallelise at the natural seam

Per-company research is embarrassingly parallel: companies don't interact. So the architecture parallelises *there* — N subagents, one message — and is sequential everywhere else (preflight, merge, synthesis).

**Why:** parallelism is a tax on debuggability. Pay it only where the speed-up matters. Per-company is the right seam: 8 companies in 5 minutes vs. 40 minutes. Trying to parallelise the merge or the summary would be premature and break determinism.

**How to apply:** find the seam where work is independent *by nature*. Parallelise there, sequential everywhere else. If you can't explain why two units of work are independent in one sentence, don't parallelise them.

### Rule 5 — Fixed templates beat clever generation

Every brief uses the same template per depth. Every summary uses the same template. The skill prompt enforces it; the agent doesn't get to improvise sections.

**Why:** stable templates make diffing trivial — humans and machines can spot what changed at a glance. Variable structure means you'd need to write a structural diff tool just to see what's new. The template is the diff contract.

**How to apply:** when output will be re-read, re-rendered, or compared over time, fix the structure first. Let creativity live inside the cells, never in the table layout.

### Rule 6 — Each skill is a self-contained executable spec

`SKILL.md` files are not documentation about a skill — they *are* the skill. Hard rules at the top, preflight, the core loop, post-processing, failure modes, "what this skill does NOT do." A subagent prompt is built from the spec; a human reading the spec sees the same logic.

**Why:** an agent re-reading the spec on each invocation needs everything in one place. Splitting context across files is how skills drift, hallucinate, and silently change behaviour. One file, fully self-contained, with explicit non-goals.

**How to apply:** when you write a spec for an AI to execute, write it as if there is no other documentation — because for the agent at execution time, there isn't.

### Rule 7 — Make non-goals explicit

The PRD has a non-goals section. Each `SKILL.md` has a "What this skill does NOT do" section. The research skill explicitly says it will not edit `watchlist.json`, will not render the dashboard, will not email anything.

**Why:** scope creep is the default failure mode of agentic systems. They will *try* to be helpful. Explicit non-goals are the only thing that keeps them in their lane.

**How to apply:** for every component, write down what it must not do. "Render briefs as a website (that's the dashboard skill)." This is the cheapest correctness mechanism in the system.

### Rule 8 — Fail loud, recover idempotently

The research skill refuses to overwrite an existing day's folder without an explicit user choice (`re-run all` / `re-run only failed/missing` / `abort`). A failed subagent leaves a stub file with the failure reason, doesn't block the others. Re-runs are safe.

**Why:** silent overwrite is the worst failure mode in a research workflow — you lose yesterday's evidence with no warning. Loud-and-recoverable is strictly better than quiet-and-fragile.

**How to apply:** every destructive operation gets an explicit confirmation step. Every operation that *could* be re-run *must* be safe to re-run. If you can't make it idempotent, you haven't finished designing it.

### Rule 9 — The seam between human and agent is structured input

`setup-watchlist` uses `AskUserQuestion` at every step. The user never types free-text into a config field that has a constrained answer. The research skill uses `AskUserQuestion` for the rerun-or-abort decision.

**Why:** free-text input is where agentic systems leak. Hallucinations, misinterpretations, and bad config all enter through the unstructured edge. Constrained input is a typed contract between user and machine.

**How to apply:** every config decision should be a multiple-choice question with a default. Save free-text for the genuinely open questions (focus areas, lens) where structure would lose information.

### Rule 10 — Designed surface and honest surface

The dashboard (M4+) is the designed surface — what someone sees if they want polish. The file tree (`briefs/`) is the honest surface — what's *actually there*. Both read the same data.

**Why:** the designed surface earns trust by being beautiful. The honest surface earns trust by being inspectable. Together they're stronger than either alone — a sceptical user can fall back to grepping the files.

**How to apply:** never let your designed surface be the only way to inspect state. Keep the underlying data in a form a power user can read directly. This is how local-first systems earn loyalty that hosted SaaS can't.

---

## 7. Applying these rules to the next complex system

When you start the next agentic build, walk this checklist before writing any code:

1. **What's the smallest substrate?** List every infra layer you're tempted to add. Cross out each one until removing it would make the milestone impossible. Build with what's left.
2. **What are the loops?** Identify the time horizons (interactive, recurring, on-demand). Each gets its own component, talking to the others only through files.
3. **What's the contract?** Define the file shapes before the skills. The data model is the architecture; everything else is convention.
4. **Where's the parallel seam?** Find the unit of work that's independent by nature. Parallelise only there.
5. **What are the templates?** Anything that will be re-read or compared over time gets a fixed template, enforced in the skill prompt.
6. **What are the non-goals?** Write them down per component before the goals. They're load-bearing.
7. **Where's the structured-input boundary?** Every constrained decision is `AskUserQuestion`. Free-text is reserved for the irreducibly open.
8. **What's the failure story?** For each operation: what does it look like when it fails? How does the next run recover? If you don't have an answer, you don't have a design.

If the answers fit on one page, the system will probably work. If they don't, the design is too complex — go back and remove things until they do.

---

## 8. What this doc is NOT

- A getting-started guide. Read the [PRD](market-research-prd.md) for that.
- A tutorial on Claude Code primitives. Read the official docs.
- A finished system. M4 (dashboard) and M5 (design sprint) are not built yet — Loop C currently has only the file-tree surface and the SessionStart hook.
- A general architecture pattern for *all* AI systems. It's a pattern for *local-first, single-user, file-substrate* systems. A multi-tenant SaaS rewrites half of these rules.

---

*If you're reading this in 6 months and something doesn't match the code, the code is right and the doc is stale. Update it.*
