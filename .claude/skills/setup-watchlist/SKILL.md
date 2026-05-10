---
name: setup-watchlist
description: Configures the daily market research watchlist via structured Q&A — captures companies to track, the lens to track them through, analysis dimensions, focus areas, schedule, and brief depth, then writes watchlist.json at the project root. Use whenever the user says "set up my watchlist", "configure research", "add competitors to track", "what should I analyse", "edit my watchlist", "change tracked companies", or starts the market research product for the first time. Re-running edits the existing config rather than overwriting blindly.
---

# Setup Watchlist

Goal: produce a complete `watchlist.json` at the project root by walking the user through a tight, structured Q&A. This is the *only* way the watchlist gets configured — no free-text watchlist editing in chat.

## Hard rules

- **Use the `AskUserQuestion` tool at every step.** Never free-prompt the user when a constrained answer will do. The whole point of this skill is structure.
- **Pre-fill sensible defaults.** A user accepting every default should finish in under 60 seconds.
- **Never overwrite an existing `watchlist.json` without showing the diff first** and getting a yes/no.
- **Don't research the companies during setup.** That's `/research`'s job. This skill is config only.
- **Cap initial setup at 10 companies.** More is allowed but warn that research time scales linearly with the count.

## Before you start

1. Check whether `./watchlist.json` exists at the project root.
2. If it exists, load it and tell the user the current state in one line:
   *"You're tracking 6 companies across 7 dimensions, daily at 08:00. What do you want to change?"*
   Then offer choices via AskUserQuestion: `add companies`, `remove companies`, `change dimensions`, `change schedule`, `start over`, `cancel`.
3. If it doesn't exist, run the full setup (steps 1–5 below).

## Steps

### 1. Context (2 questions)

Ask via AskUserQuestion:

**Q1. What's the tracker's role?** (single-select with an "other" free-text fallback)
- PM (product / strategy)
- Founder / exec
- Investor / analyst
- BD / partnerships
- Other (specify)

**Q2. What's the purpose of this watchlist?** (single-select)
- `competitive_intel` — competitors I'm benchmarking against
- `partner_watch` — ecosystem / partner moves
- `acquirer_watch` — potential acquirers or strategic buyers
- `market_scan` — broad market awareness, no fixed lens

These frame the brief — the same company reads differently through each lens. Don't skip.

### 2. Companies (loop)

Ask for a comma-separated list of companies, then for each one, capture via AskUserQuestion:

- **Name** — as given.
- **Primary domain** — auto-suggest from the name (e.g. `Stripe → stripe.com`), confirm with the user. If unknown, accept `""` and move on.
- **Relationship** (single-select): `competitor`, `partner`, `acquirer`, `peer`, `wildcard`.
- **Focus areas** — open free-text, 0–3 short phrases. *"What specifically should we watch at this company?"* (e.g. "stablecoin rails", "EU regulatory posture", "enterprise pricing"). This is the only step that's intentionally open-ended.

If the user lists >10 companies, warn once: *"Research time scales linearly. 10 is the comfortable cap; 15+ will start missing the 08:00 window. Continue anyway?"* Accept their answer; don't refuse.

### 3. Dimensions (1 question + 1 follow-up)

Show the default dimension set and ask via AskUserQuestion (multi-select, all pre-checked):

**Default ON:**
- `news` — News & announcements
- `product` — Product / feature launches
- `hiring` — Hiring & org changes
- `funding` — Funding / financial signals
- `pricing` — Pricing & packaging changes
- `sentiment` — Public sentiment (X, HN, Reddit, review sites)
- `leadership` — Leadership moves

**Default OFF (offer as opt-in):**
- `regulatory` — Regulatory / legal
- `partnerships` — Partnerships & integrations
- `patents` — Patents & technical signals
- `customer_wins` — Customer wins / logos
- `oss` — Open-source activity

**Follow-up (optional):** *"Any dimension you want weighted higher than the rest?"* — single-select from chosen dimensions or "none". Weighted dimensions get `weight: 2`; the rest get `weight: 1`.

### 4. Delivery (3 questions, all with defaults)

Ask via AskUserQuestion:

- **Cadence**: `daily` (default), `weekdays`, `weekly`.
- **Local delivery time**: default `08:00`. Offer: `06:00`, `07:00`, `08:00`, `09:00`, custom.
- **Depth**:
  - `executive` — TL;DR + top 3 headlines per company. Fast.
  - `standard` *(default)* — full per-company brief template (TL;DR, news, product, hiring, funding, pricing, sentiment, "what this means").
  - `deep` — standard + analyst-style commentary and a "what to do" section.

Format is `markdown` for v1 — don't ask, just record it.

### 5. Confirm + write

Show a final summary in this exact shape (replace the bracketed bits):

```
WATCHLIST · [N] companies · [M] dimensions · [cadence] [HH:MM] · [depth] depth
LENS: [purpose]

COMPANIES
─ [Name] ([relationship]) — [focus_areas joined by comma, or "—"]
─ ...

DIMENSIONS ([weighted dim if any in caps])
[comma-separated dimension labels]
```

Then ask via AskUserQuestion: `confirm` / `edit` / `cancel`.

- `confirm` → write `./watchlist.json` per the schema below, set `updated_at` to ISO8601 now, set `version: 1`. Print a one-line confirmation: *"Saved. Run `/research` to generate today's brief."*
- `edit` → ask which step to revisit (1, 2, 3, or 4), jump back, then return here.
- `cancel` → write nothing, exit.

## Schema (write exactly this shape to `watchlist.json`)

```json
{
  "version": 1,
  "updated_at": "ISO8601 timestamp",
  "context": {
    "tracker_role": "pm | founder | investor | bd | other",
    "tracker_role_other": "string or null",
    "purpose": "competitive_intel | partner_watch | acquirer_watch | market_scan"
  },
  "companies": [
    {
      "name": "string",
      "domain": "string",
      "aliases": ["string"],
      "relationship": "competitor | partner | acquirer | peer | wildcard",
      "focus_areas": ["string"],
      "added_at": "YYYY-MM-DD"
    }
  ],
  "dimensions": [
    {
      "id": "news | product | hiring | funding | pricing | sentiment | leadership | regulatory | partnerships | patents | customer_wins | oss",
      "label": "string",
      "enabled": true,
      "weight": 1
    }
  ],
  "delivery": {
    "cadence": "daily | weekdays | weekly",
    "time_local": "HH:MM",
    "depth": "executive | standard | deep",
    "format": "markdown"
  }
}
```

## Edit-mode behavior

When `watchlist.json` already exists and the user picks an edit action:

- `add companies` → run step 2 only, append to `companies[]`, keep everything else.
- `remove companies` → AskUserQuestion multi-select over current company names, drop selected.
- `change dimensions` → run step 3 only, replace `dimensions[]`.
- `change schedule` → run step 4 only, replace `delivery`.
- `start over` → run all 5 steps, replace the whole file.

Always show the diff (added / removed / changed) before writing.

## Failure modes to handle

- **User wants something not in the schema** (e.g. "track this company on Slack mentions"). Acknowledge, log it as a `focus_area` for that company, and tell them: *"Custom signals like that aren't a first-class dimension yet — captured as a focus area. If it matters across multiple companies, we should add it to the schema."*
- **Empty answers.** If the user skips a required field, AskUserQuestion again with the default highlighted.
- **Duplicates.** If a company name already exists in the list, ask: *"already tracked — update or skip?"*
