---
name: cv-ingest
description: Ingests a CV (file path, attachment, or pasted text), parses it into a structured markdown reference, and saves it to JobMatch/cv.md so future skills can match the person against open jobs. Use whenever the user says "ingest my CV", "process my resume", "set up my CV", "update my CV reference", "load my resume", "/cv-ingest", or hands over a CV file/paste with the intent to use it for job matching. Re-running prompts before overwriting an existing cv.md.
---

# CV Ingest

Goal: turn a raw CV into a clean, structured `cv.md` at `JobMatch/cv.md` that downstream skills (job search, match, tweak, apply) can read as the canonical reference for who this person is and what they want next.

## Hard rules

- **Output location is fixed: `JobMatch/cv.md`.** Don't write anywhere else.
- **Never silently overwrite an existing `cv.md`.** If one exists, show the user a one-line summary of what's there and ask: replace, merge new info in, or cancel.
- **Don't invent facts.** If a date, title, or scope isn't in the source, leave a `TODO:` placeholder and surface it to the user — don't guess.
- **Capture forward-looking intent, not just history.** A CV alone doesn't say what jobs to match against. Always ask the targeting questions in step 3.
- **Use `AskUserQuestion` for any constrained-answer prompts.** Free-text only when the answer genuinely is open-ended (e.g. "paste deal-breakers").

## Steps

### 1. Get the CV

Ask the user how they want to provide it. Use `AskUserQuestion` with these options:
- `file path` — they'll give an absolute path (PDF, docx, txt, md)
- `paste` — they'll paste the text directly into chat
- `attachment` — they've already attached/dragged a file into this turn

Then ingest:
- **PDF**: use the `Read` tool with the file path (handles PDFs natively). For PDFs >10 pages, ask first if all of it is the CV or just specific pages.
- **docx**: invoke the `anthropic-skills:docx` skill to extract text.
- **txt / md / pasted**: read directly.

Confirm in one line what you got: *"Read 3-page PDF, ~850 words, looks like a CV — proceeding."* If it doesn't look like a CV (no work history, no name), stop and tell the user.

### 2. Parse into structure

Extract these sections. Leave any section out if the source has nothing for it — don't pad with empty headers.

- **Identity**: name, email, phone, location, LinkedIn, portfolio/site, GitHub.
- **Headline**: their own one-liner positioning if present, else a 12-word summary you write from the experience.
- **Experience** (most recent first): for each role capture
  - Company, title, dates (start–end, or `present`), location/remote
  - Scope (1 line: team size, budget, area owned)
  - 3–6 achievement bullets, kept verbatim where possible
  - Tools/tech/methods named in the role
- **Education**: degree, institution, year, honors.
- **Skills**: split into `technical`, `domain`, `leadership/soft`. Don't duplicate what's already in the experience bullets — only list things repeated across roles or explicitly highlighted.
- **Certifications & awards**.
- **Side projects, writing, speaking** — anything outside formal employment that shows range.
- **Languages**: human languages with proficiency.

### 3. Capture targeting intent (critical — don't skip)

A CV is backward-looking. Matching needs forward-looking signal. Ask via `AskUserQuestion`:

**Q1. Target seniority** (single-select)
- IC / senior IC
- Manager / lead
- Director / head of
- VP+
- Founder / 0→1
- Open / mixed

**Q2. Role types** (multi-select)
- Product management
- Product strategy / ops
- Founding PM / 0→1
- Growth / monetisation
- Platform / infra PM
- Technical PM / AI PM
- Other (specify)

**Q3. Domains of interest** (multi-select, plus "other" free-text)
- Pre-fill from the experience you parsed (e.g. travel, fintech, dev tools) and let them add/remove.

**Q4. Geography & work mode** (single-select)
- Remote-only
- Hybrid (specify city)
- On-site (specify city)
- Open

**Q5. Deal-breakers** (free text, one line)
- Examples to prompt: *"no crypto, no defence, no Series A or earlier, EU timezone only, base ≥ €X."*

**Q6. What you're optimising for next** (free text, one line)
- *"Bigger scope", "AI-native company", "back to 0→1", "stability + comp"* — whatever they'd actually say out loud.

### 4. Write `JobMatch/cv.md`

Use this layout exactly. Keep it human-readable; downstream skills will parse the headings.

```markdown
# CV — {Name}

_Last updated: {YYYY-MM-DD}_

## Identity
- Email: …
- Location: …
- LinkedIn: …
- (other links)

## Headline
{one-line positioning}

## Targeting (forward-looking)
- Seniority: …
- Role types: …
- Domains: …
- Geography / mode: …
- Deal-breakers: …
- Optimising for: …

## Experience

### {Title} — {Company}  ({start} – {end})
- Scope: …
- {achievement bullet}
- {achievement bullet}
- Tools: …

(repeat per role)

## Skills
- **Technical**: …
- **Domain**: …
- **Leadership / soft**: …

## Education
- …

## Certifications & awards
- …

## Side projects, writing, speaking
- …

## Languages
- …

## Open questions / TODOs
- {anything you flagged as missing from the source}
```

### 5. Confirm and exit

Print a 3-line summary:
- Path written: `JobMatch/cv.md`
- Roles parsed: N
- Open TODOs: M (list them)

Tell the user the next skill in the chain will read this file — don't describe how. End.

## On re-runs

If `JobMatch/cv.md` already exists when the skill is invoked:
1. Read it.
2. Show: *"Existing cv.md last updated {date}, {N} roles, targeting {seniority} in {domains}."*
3. Ask via `AskUserQuestion`: `replace entirely`, `add a new role`, `update targeting only`, `fix a specific section`, `cancel`.
4. Branch accordingly — don't redo the full flow if they just want to update one field.
