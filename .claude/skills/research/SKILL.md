---
name: research
description: Runs the daily market research playbook against the configured watchlist — spawns one parallel subagent per company, each producing a structured brief, then synthesises a cross-cutting summary. Outputs land in briefs/YYYY-MM-DD/. Use whenever the user says "/research", "run research", "generate today's brief", "do the daily run", "refresh the watchlist", or when the daily schedule fires. Reads watchlist.json — fails fast if it doesn't exist (route to setup-watchlist instead).
---

# Research

Goal: produce a complete set of dated Markdown briefs (one per company + one cross-cutting summary) for today's watchlist run, using parallel subagents for the per-company work.

## Hard rules

- **Never invent facts.** Every concrete claim in a brief must have a source URL. Unsourced bullets get cut, not softened.
- **Every subagent runs in parallel.** Spawn all N company-research agents in a single message with multiple `Agent` tool calls. Sequential is wrong — the whole point of this design is the speed-up.
- **Time-window every search.** Default lookback: last 48 hours. Ignore older items unless they're the *only* signal and the user's depth is `deep`.
- **Output template is fixed per depth.** Don't improvise sections. Stable templates make the day-over-day diff trivial; freelancing breaks it.
- **Never overwrite an existing brief without a flag.** If today's folder already exists, ask: re-run, append, or abort. Default abort.
- **One brief per company file.** Path: `briefs/YYYY-MM-DD/{CompanyName}.md`. Plus `briefs/YYYY-MM-DD/_summary.md` for the cross-cutting view.

## Preflight

1. Read `./watchlist.json`. If missing → tell the user to run the `setup-watchlist` skill first and stop.
2. Validate: `companies[]` non-empty, `dimensions[]` has at least one `enabled: true`, `delivery.depth` is one of `executive | standard | deep`.
3. Today's date in user's local time (use the date from system context; don't ask). Set `TODAY = YYYY-MM-DD`.
4. Check `briefs/{TODAY}/` — if it exists with files in it, ask via AskUserQuestion: `re-run all (overwrite)`, `re-run only failed/missing`, `abort`.
5. Read `briefs/_seen.json` if it exists (long-term dedupe memory). Read yesterday's `briefs/{YESTERDAY}/{Company}.md` per company if it exists (for diffing).
6. `mkdir -p briefs/{TODAY}/`.

## Spawn parallel subagents (the core)

For each company in `watchlist.companies`, spawn **one `Agent` call with `subagent_type: general-purpose`** in a single message. All companies run simultaneously. (Explore is the wrong choice — it can't write files.)

Each subagent gets a **self-contained prompt** built from the template below. Subagents have no memory of this conversation, so the prompt must include everything they need: company info, dimensions, depth, yesterday's brief (inlined), output template, output path, source rules.

### Subagent prompt template

Fill the `{{...}}` placeholders before sending. Don't deviate from this structure.

```
You are a market-research analyst. Produce a brief on ONE company by today's deadline.

COMPANY
- Name: {{name}}
- Primary domain: {{domain}}
- Aliases: {{aliases joined by comma, or "—"}}
- Relationship to the user: {{relationship}}  (one of competitor / partner / acquirer / peer / wildcard)
- Focus areas (what specifically to watch): {{focus_areas joined by comma, or "—"}}

USER CONTEXT
- Tracker role: {{context.tracker_role}}
- Lens: {{context.purpose}}  (frame all analysis through this lens — the same fact reads differently per lens)

DIMENSIONS (only cover these, in this order, with enabled=true)
{{for each enabled dimension: "- {{label}} (weight {{weight}})"}}

DEPTH: {{delivery.depth}}
- executive → only TL;DR (≤3 bullets) and top 3 headlines per dimension. Skip "what this means."
- standard → full template (all enabled dimension sections + "what this means for us" with 2 bullets).
- deep → standard + a final "Analyst take" section (≤200 words, opinionated, references the user's lens) + a "Watch next" list of ≤3 trigger events.

LOOKBACK
- Time window: last 48 hours. Surface older items only if they're the only signal AND depth=deep.

YESTERDAY'S BRIEF (for diffing — flag what's NEW vs. what was already covered)
{{inline yesterday's brief markdown, or "no prior brief"}}

ALREADY-SEEN ITEMS (don't re-surface these unless materially updated)
{{inline relevant entries from briefs/_seen.json, or "none"}}

RESEARCH METHOD
1. Use WebSearch first to find candidate sources. Prefer: company official channels (blog, newsroom, GitHub releases, status page, jobs page), Tier-1 press (Reuters, Bloomberg, FT, WSJ, TC), authoritative trade press in this domain, regulatory filings.
2. Use WebFetch on each promising URL to read the actual content. Don't quote from search snippets alone — they hallucinate.
3. Triangulate: for any non-trivial claim, prefer ≥2 independent sources. If only one source, mark the bullet "[single-source]".
4. Skip low-quality sources: SEO farms, AI-generated news aggregators, anonymous Substacks, Reddit comments without primary links.

WRITING RULES
- Every concrete claim has a source URL inline. No URL → cut the claim.
- Bullets are facts, not vibes. ≤25 words each. Numbers and dates beat adjectives.
- No headlines like "interesting news" — write the actual development.
- If a dimension has nothing material, write the section header and "No material signal in the last 48h."
- Tag NEW items vs. yesterday with a leading `[NEW]`. Tag updated items with `[UPDATED]`.

OUTPUT
Write the brief in the EXACT template below to: ./briefs/{{TODAY}}/{{name}}.md

—— template start ——
# {{name}} — {{TODAY}}

**Domain:** {{domain}} · **Relationship:** {{relationship}} · **Lens:** {{purpose}}

## TL;DR
- (≤3 bullets, the most important things from the last 48h, each with a source URL)

## News & announcements
- ...

## Product / feature launches
- ...

(... one section per enabled dimension, in the order listed above ...)

## What this means for us
*(2 bullets, opinionated, framed by the user's lens. Skip if depth=executive.)*
- ...
- ...

## Analyst take
*(only if depth=deep — ≤200 words.)*

## Watch next
*(only if depth=deep — ≤3 specific trigger events that would change the call.)*
- ...

## Sources
- (deduplicated list of every URL cited above, with publish date)
—— template end ——

RETURN to the parent agent (do NOT include in the file): a short JSON object with:
- new_items: array of {dimension, headline, url} for each [NEW] bullet (used to update _seen.json)
- failures: array of {dimension, reason} for any dimension you couldn't research (rate limit, no signal, etc.)
- file_path: the path you wrote to
```

## After all subagents return

1. **Update seen-memory.** Merge each subagent's `new_items` into `briefs/_seen.json`. Keep last 30 days; prune older entries.
2. **Compose the cross-cutting summary.** Read all briefs you just wrote. Produce `briefs/{TODAY}/_summary.md`:

```
# Watchlist summary — {{TODAY}}

**Coverage:** {{N companies}} · **Lens:** {{purpose}} · **Run depth:** {{depth}}

## Today's call (top 3)
*The 3 things from across the watchlist that most demand attention. Each cites which companies it touches.*
1. ...
2. ...
3. ...

## Cross-cutting themes
*Up to 3 themes that recur across ≥2 companies. Each: a 1-line headline, a 2-line body, the companies it covers.*
- ...

## Per-company status (one line each)
- **{{Name}}** — {{badge}} — {{TL;DR line 1}}
  - badge: HOT (≥3 NEW items), NEW (1–2 NEW items), QUIET (0 NEW items)

## Diff vs. yesterday
- Net-new items: N
- Items closed/resolved: N
- New themes: N

## Sources
- (deduplicated across briefs, total count)
```

3. **Print to chat** a 5–10 line text summary so the user sees the result without opening a file:
   - Date · companies covered · NEW items count · failures count
   - Top 3 from the cross-cutting summary, verbatim
   - File paths for the per-company briefs
4. **If any subagent failed**, list which company / which dimensions, and offer to re-run just those.

## Failure handling

- **Single subagent fails entirely.** Note the company in the chat summary, leave a stub `briefs/{TODAY}/{Company}.md` with `# {Company} — RESEARCH FAILED · {timestamp} · {reason}`. Don't block other companies.
- **Watchlist empty.** Refuse, route to `setup-watchlist`.
- **Network / web-tool errors.** Subagents handle their own retries; if a subagent gives up, treat as the previous case.
- **Rate-limit / quota.** If multiple subagents fail with the same rate-limit reason, stop spawning new ones, finish what's running, and tell the user to retry in N minutes. Don't silently swallow.

## Idempotency & re-runs

- Re-running on the same day with `re-run all` overwrites the day's folder.
- `re-run only failed/missing` reads `briefs/{TODAY}/`, identifies stubs and missing files, spawns subagents only for those.
- `_seen.json` is append-only across the day — don't reset it on re-runs.

## What this skill does NOT do

- Edit `watchlist.json` (that's `setup-watchlist`).
- Render the briefs as a website (that's the dashboard / `/dashboard` skill, not yet built).
- Email or message the briefs anywhere — local files only.
- Cross-day analysis beyond the 1-day diff (a separate `/research-trend` skill could do this later).
