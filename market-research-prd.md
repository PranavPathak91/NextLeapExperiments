# PRD — Daily Market Research Agent (Claude Code, Local)

**Owner:** Pranav
**Status:** Draft v0.2
**Last updated:** 2026-05-10
**Purpose of this doc:** Spec a small, demo-able product built entirely on Claude Code primitives — no hosting, no managed infra, no extra API keys. It is a teaching artifact for a PM training on AI / Claude Code, so the PRD doubles as a map between *product features* and *Claude Code building blocks*. v0.2 adds a local web frontend so we can run an **MVP design sprint** on top of the same data.

---

## 1. Problem & motivation

PMs, founders, and strategy folks need a recurring read on a watchlist of companies (competitors, partners, acquirers, peers in an adjacent space). Today this is one of three bad options:

- Manual: skim Google News + LinkedIn + Crunchbase every morning, dies after week one.
- Newsletters: shallow, one-size-fits-all, can't be scoped to *your* watchlist.
- SaaS tools (Crayon, Klue, Similarweb): expensive, slow to set up, overkill for personal/team use.

We want something that costs ~zero, sets up in five minutes, and delivers a tight daily brief on a watchlist the user controls — running entirely inside Claude Code on the user's laptop.

## 2. Goals

- A user can define a watchlist of N companies in <2 minutes.
- The agent produces a daily Markdown brief per company + one cross-cutting summary, surfaced inside Claude Code.
- Brief covers: news, product launches, hiring signals, funding, leadership moves, pricing changes, public sentiment.
- Runs on a schedule with no human in the loop, but is also runnable on-demand.
- **A local website renders the briefs as a browsable dashboard** — opened with one command, reading directly from the same Markdown files.
- Lives entirely in the local filesystem. No hosted backend, no DB, no third-party API keys, no auth. Web frontend is served by a local dev server (e.g. Vite) on `localhost`.

## 3. Non-goals

- Multi-user collaboration, shared hosting, auth, or a deployed web app. The frontend is local-only.
- Persisting structured data in a queryable DB (Markdown files are the database).
- Replacing real CI tools like Crayon — this is a personal/research tool.
- Investment-grade analysis or financial recommendations.
- Bypassing paywalls or scraping sites that disallow it.

## 4. Target users

Primary: PMs and founders building or competing in a defined market, who already use Claude Code.
Secondary (and the actual use case here): the **PM training cohort** — the product is a teaching vehicle for showing how Claude Code primitives compose into a real workflow.

## 5. User stories

1. *As a PM*, I run `/watchlist add Stripe Adyen Checkout.com` and have a tracked watchlist.
2. *As a PM*, I run `/research` and get a fresh brief on every company within ~5 minutes.
3. *Each morning*, a brief is generated automatically and surfaced when I open Claude Code.
4. *As a PM*, I open `briefs/2026-05-10/Stripe.md` and see a structured one-pager.
5. *As a PM*, I run `/research-diff Stripe` and see what changed vs. yesterday.
6. *As a PM*, I run `/dashboard` and a browser tab opens at `localhost:5173` showing today's briefs as a polished, browsable site.
7. *As a PM in a design sprint*, I screenshot the dashboard, ask Claude to redesign the company-card layout, see the change in the browser within seconds, and iterate.

## 6. Functional requirements

### 6.1 Watchlist management
- Stored as `watchlist.json` in the project root.
- Each entry: `{ name, domain, aliases[], focus_areas[] }`.
- Managed via slash commands: `/watchlist add`, `/watchlist remove`, `/watchlist show`.

### 6.2 Daily research run
- Triggered on a schedule (default: 08:00 local) or manually via `/research`.
- For each company, an Explore subagent runs the research playbook in parallel.
- Output: `briefs/YYYY-MM-DD/{Company}.md` per company + `briefs/YYYY-MM-DD/_summary.md`.

### 6.3 Per-company brief structure
Fixed template so diffs are stable across days:
1. TL;DR (≤3 bullets)
2. News & announcements (last 24–48h)
3. Product / feature launches
4. Hiring & org changes
5. Funding / financial signals
6. Pricing & packaging changes
7. Public sentiment (X, HN, Reddit, review sites — read-only, summarized)
8. "What this means for us" — 2 bullets, opinionated
9. Sources

### 6.4 Cross-cutting summary
- One-page roll-up: top 5 things across the watchlist, themes, and any "act today" items.

### 6.5 Diff & memory
- Each run reads yesterday's brief from disk and flags net-new items only.
- Long-term memory of "things we've already seen" stored in `briefs/_seen.json` to avoid repeats.

### 6.6 Local web dashboard
- A minimal Vite + React (or plain HTML + a tiny markdown renderer) app in `web/` that reads the `briefs/` directory at build/dev time.
- Started by `/dashboard` slash command, which runs `npm run dev` in the background and opens `localhost:5173`.
- Pages:
  1. **Today** — grid of company cards with TL;DR + "what's new" badge.
  2. **Company detail** — full brief rendered from the Markdown file.
  3. **Timeline** — historical briefs for one company across days.
  4. **Themes** — cross-cutting summary view.
- Reads Markdown directly. No build step beyond Vite's hot reload. No backend.
- The dashboard is a *first-class output*, not an afterthought — this is the surface the design sprint iterates on.

## 7. Technical design — Claude Code primitives

This is the demo-relevant part. Every requirement maps to something the cohort can see in Claude Code:

| Product capability | Claude Code primitive |
|---|---|
| User input for watchlist | **Slash commands** (`/watchlist`, `/research`) defined in `.claude/commands/` |
| Parallel research per company | **Subagents** (`Explore` / general-purpose), one Agent call per company in a single message |
| Daily scheduled run | **Scheduled tasks** (the `schedule` skill / cron) firing `/research` at 08:00 |
| Web research | **WebSearch + WebFetch** tools (no API key needed) |
| Storage | **Local Markdown files** under `briefs/`, plus one `watchlist.json` |
| Memory across runs | **Files on disk** (`briefs/_seen.json`, prior-day briefs) — no DB |
| Surfacing the brief on open | **SessionStart hook** in `.claude/settings.json` that prints today's `_summary.md` |
| Guardrails / formatting | **CLAUDE.md** at repo root with the brief template + research rubric |
| Optional: review the brief | **Custom subagent** (`brief-reviewer`) that critiques the draft before saving |
| Local web dashboard | **Vite dev server** spawned by a `/dashboard` slash command, reading `briefs/` |
| Visual iteration in the design sprint | **Claude Preview MCP tools** (`preview_screenshot`, `preview_click`, `preview_resize`) so Claude can *see* its own UI and iterate without a human screenshotting |
| Design system handoff (optional) | The `design:design-system` and `design:design-critique` skills, run against the live dashboard |

No API keys, no hosting, no external services beyond what Claude Code already does.

## 8. Demo workflow (the actual training arc)

The PRD is also the lesson plan. Two arcs, run back-to-back:

### Arc A — Build the agent (~60 min)
1. **Start from zero.** Empty folder. Show `CLAUDE.md` as the agent's "operating manual."
2. **Add a slash command.** `/watchlist add` — show how a 20-line markdown file becomes a feature.
3. **Add a subagent.** `research-company` — show the prompt, the tools, and one-shot it.
4. **Parallelize.** Run all companies in one message → show the speed-up vs. sequential.
5. **Add storage.** Briefs land on disk, structured by date.
6. **Add a hook.** SessionStart prints today's summary — the product now feels alive.
7. **Add a schedule.** Use the `schedule` skill so it runs at 08:00 even when no one types anything.
8. **Add a reviewer subagent.** Show how a second agent can critique the first → quality lift.

### Arc B — MVP design sprint on the dashboard (~60 min)
This is the new arc. Once briefs exist on disk, we use them as raw material for a live design sprint inside Claude Code.

1. **Scaffold.** Ask Claude to spin up a Vite + React app in `web/` that reads `briefs/`. Wire `/dashboard` to start it.
2. **First render.** Open `localhost:5173`. It will look ugly — that's the point.
3. **Brief the design.** Paste a 3-line product brief ("PMs scanning their watchlist over coffee, mobile-friendly, calm and dense, not flashy") into chat.
4. **Generate options.** Ask Claude to produce 3 distinct visual directions (e.g. "FT-style serif," "Linear-style minimal," "Bloomberg terminal"). Each is a branch of `web/` styles.
5. **Critique loop.** Use the `design:design-critique` skill against screenshots from `preview_screenshot`. Claude critiques its own output.
6. **Pick + refine.** Choose a direction. Iterate on hierarchy, density, typography. Each change is visible in the browser within seconds.
7. **Accessibility pass.** Run `design:accessibility-review` against the live page — fix contrast and focus issues.
8. **Handoff artifact.** Run `design:design-handoff` to generate a spec doc from the final design — a tangible take-home.

The cohort sees a real "PRD → working agent → designed UI → spec sheet" loop in one sitting, all in Claude Code, all local.

Each step is ~5–8 minutes and produces a working artifact. The PRD-to-primitive table in §7 is the cheat sheet.

## 9. Success metrics

For the *training* (what we're actually optimizing):
- ≥80% of attendees can name the core primitives used (slash command, subagent, hook, schedule, local preview).
- ≥50% of attendees ship a modified version (different watchlist, different brief template, or restyled dashboard) by end of session.
- ≥1 attendee runs a full design sprint loop (brief → 3 options → critique → pick → refine) on their own watchlist before the session ends.

For the *product* (if anyone keeps using it):
- Daily brief generated on ≥5 of 7 days in the first week without manual intervention.
- User opens the local dashboard ≥3x/week.

## 10. Risks & mitigations

- **Web search quality drift.** WebSearch results vary; mitigate with explicit source-quality rubric in `CLAUDE.md` and a reviewer subagent.
- **Rate limits / quotas.** Batch per-company research; cap watchlist at ~10 companies for the demo.
- **Hallucinated facts.** Every claim in a brief must include a source URL; reviewer subagent rejects unsourced claims.
- **Schedule reliability.** Local cron only fires when the laptop is awake — acceptable for a demo, document the limitation.
- **Stale briefs / repeats.** `_seen.json` dedupe + diff-based summary.
- **Dashboard rabbit-hole in the design sprint.** Easy to spend the whole afternoon on CSS. Mitigate with a hard time-box per step and a "good enough" rubric — the goal is to demo the *loop*, not ship a finished design.
- **Node / Vite setup friction.** Some attendees may not have Node installed. Provide a one-liner check at the start of Arc B; offer a plain HTML + vanilla JS fallback for anyone stuck.

## 11. Milestones

- **M0 — Skeleton (30 min):** `CLAUDE.md`, `watchlist.json`, `/watchlist` command, manual `/research` for one company.
- **M1 — Parallel + template (30 min):** Subagent, fixed brief template, all companies in parallel, files on disk.
- **M2 — Daily loop (20 min):** SessionStart hook + scheduled task firing `/research` at 08:00.
- **M3 — Reviewer + diff (20 min):** Reviewer subagent, diff vs. yesterday, cross-cutting summary.
- **M4 — Dashboard scaffold (20 min):** Vite + React app in `web/`, `/dashboard` command, basic Today view reading from `briefs/`.
- **M5 — Design sprint (40 min):** 3 visual directions, critique loop with `preview_screenshot`, pick + refine, a11y pass, handoff doc.

Total build time live: ~160 minutes, splits naturally into a morning (Arcs A, M0–M3) and afternoon (Arc B, M4–M5).

## 12. Open questions

- Should the brief format be customizable per company (e.g. different template for an acquirer vs. a competitor), or fixed for v1? *Lean: fixed for v1, configurable in v2.*
- Do we want a "watchlist suggestion" command that proposes companies based on a one-line market description? *Nice demo, defer.*
- How do we handle companies with very low public signal (stealth startups)? *Brief should explicitly say "no signal today" rather than hallucinate.*
- Should the dashboard include a "compare two companies side-by-side" view? *Good design-sprint exercise but not core to v1 — leave as a stretch goal.*
- Plain HTML vs. Vite + React for the dashboard? *Lean Vite + React for the demo since hot reload + component refactors make the design iteration loop more satisfying. Plain HTML is the fallback.*
