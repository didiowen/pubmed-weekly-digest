---
name: pubmed-weekly-digest
description: |
  Weekly medical-literature digest agent — searches the past 7 days on PubMed
  across a configurable list of journals, fills in missing abstracts via CrossRef,
  writes a TL;DR and a one-line Hot Take for each article (English by default;
  language is configurable in Step 4), and renders a Markdown digest to a
  local output directory.

  Trigger when the user says "跑週報", "weekly journal", "weekly digest",
  or otherwise asks to run a weekly journal summary.
---

# PubMed Weekly Digest

> A Claude Code skill: searches the past 7 days on PubMed across a configurable
> list of journals, refills missing abstracts via CrossRef, writes a TL;DR and
> a one-line Hot Take per article, and renders a Markdown digest to a local
> output directory. Committing or publishing the digest is up to you.

## Configuration

Edit these values before running the skill in a new vault:

| Key                | Default                                         | Notes                                              |
|--------------------|-------------------------------------------------|----------------------------------------------------|
| `OUTPUT_DIR`       | `+/journals`                                    | Where `{week_label}.md` is written                 |
| `JOURNALS`         | See Step 1 table                                | The list of `[Journal]` queries — customise freely |
| `CROSSREF_MAILTO`  | `pubmed-weekly-digest@noreply.example`          | CrossRef polite-pool identifier (any valid email)  |

`CROSSREF_MAILTO` can also be supplied via an environment variable
`UNPAYWALL_EMAIL` if you prefer not to bake it into the file. If neither is set,
CrossRef calls still go out (some endpoints just rate-limit harder).

See `README.md` (same folder) for setup details and a step-by-step guide to
swapping the journal list.

## ⚠️ ISO week calculation (read this first)

This skill runs at the start of a week and covers the **just-ended** week.

```python
from datetime import date, timedelta

today = date.today()

# Search window: past 7 days (excluding today)
date_to   = today - timedelta(days=1)   # yesterday
date_from = today - timedelta(days=7)   # 7 days ago

# Week label: use the ISO week of the just-ended week, NOT today's week
target_date = today - timedelta(days=7)
iso_year, iso_week, _ = target_date.isocalendar()
week_label = f"{iso_year}-W{iso_week:02d}"   # e.g. 2026-W19

filepath = f"{OUTPUT_DIR}/{week_label}.md"
```

**Example**: today is 2026-05-11 (W20) → `week_label = 2026-W19`, search range 2026-05-04 to 2026-05-10.

> **Pitfall**: do NOT use `today.isocalendar()` for the label — that mis-labels
> W19 content as W20.

---

## Step 1 — PubMed search

Call `mcp__PubMed__search_articles` (datetype=`edat`) **in parallel** for each
journal. The default list is six ID/haematology journals; replace freely:

| Query string                          | max_results |
|---------------------------------------|-------------|
| `Clin Infect Dis[Journal]`            | 50          |
| `Emerg Infect Dis[Journal]`           | 50          |
| `MMWR Morb Mortal Wkly Rep[Journal]`  | 30          |
| `Transpl Infect Dis[Journal]`         | 50          |
| `Blood[Journal]`                      | 50          |
| `N Engl J Med[Journal]`               | 50          |

**date_from** = `date_from` (YYYY/MM/DD), **date_to** = `date_to` (YYYY/MM/DD)

**PubMed query caveats** — these apply to every entry in the table:

- The MCP wrapper rejects **wildcards** (`mycobacteri*` → `INVALID_PARAMETERS`).
  Expand with `OR` instead: `mycobacterium OR tuberculosis`.
- The **boolean-operator cap is 20**. A single query like `(a OR b OR ... OR u)`
  fails; split into two parallel queries if needed.
- Add a topic filter with `AND (term1 OR term2)` to high-volume generals,
  e.g. `N Engl J Med[Journal] AND (infection OR sepsis OR antimicrobial)`.

**If search fails (MCP error or 0 articles for all journals): stop and report.**

---

## Step 2 — Fetch full metadata

Call `mcp__PubMed__get_article_metadata` in parallel batches (one batch per
journal). Each batch is **≤ 20 PMIDs** — the MCP silently truncates beyond that.

---

## Step 2.5 — CrossRef abstract refill

Scan all articles. If `abstract` is empty (or `"[Abstract not available]"`,
which PubMed sometimes returns as a string) and a `doi` is present, query
CrossRef:

- Endpoint: `https://api.crossref.org/works/{urlencoded_doi}?mailto={CROSSREF_MAILTO}`
- User-Agent: `pubmed-weekly-digest/1.0`
- Timeout: 15 seconds; cache by DOI within one run.
- If `message.abstract` is present in the response: strip JATS/HTML tags and
  collapse whitespace, then write back to the article's `abstract` field.
- If CrossRef returns no abstract, non-200, timeout, or parse failure:
  do NOT abort. Keep the article in a "still missing abstract" list.

**Articles still missing abstracts** after CrossRef:

- **Interactive mode**: list the affected articles (PMID, title, DOI) and ask
  the user to paste in abstracts manually, then continue. If the user says
  skip, leave `TL;DR` blank and note `(no abstract)` in the final output.
- **Autonomous mode**: leave `TL;DR` blank for those articles and list the
  PMIDs in the final report so the user can backfill on a future run.

---

## Step 3 — Filter articles

Drop articles where `article_types` contains:

- `"Erratum"` or `"Published Erratum"`
- `"Editorial"` or `"Comment"` (no abstract → no meaningful annotation)

A section with zero surviving articles is fine — show a placeholder in Step 5.

### Optional: per-journal type filtering

The default config includes an example for N Engl J Med where only Original
Articles and Reviews are kept:

- `article_types` contains `"Journal Article"` but NOT `"Case Reports"`,
  `"Editorial"`, `"Comment"`, `"Letter"`, `"News"`, `"Biography"`,
  `"Historical Article"`, `"Portrait"`, `"Interview"`, `"Personal Narrative"`
  → treat as Original Article
- `article_types` contains `"Review"` or `"Systematic Review"`
  → treat as Review Article
- Anything else from N Engl J Med → drop from this section

Adjust or remove this block for your own journal mix.

---

## Step 4 — TL;DR and Hot Take annotation

For each surviving article:

- **TL;DR**: a 1–2 sentence English summary covering study design, key
  findings, and clinical implications. Keep drug names and pathogen names
  (bacterial / viral / fungal) in their canonical form; do not translate the
  article title.
- **Hot Take**: one short humorous English sentence per article. Pick a register
  that fits the content:
  - *Positive* — for hopeful findings, breakthroughs, good outcomes
    (excited / cheerful tone)
  - *Snarky* — for bad news, hard pathogens, depressing epidemiology, or old
    problems with no new answers (deadpan / self-aware tone, not cruel)

If `abstract` was never filled in, leave both fields blank.

> **Change the output language**: this step is pure prompting. Replace
> "English" above with "Traditional Chinese", "Spanish", "Japanese", or
> whatever you prefer — Claude will translate the source abstracts on the fly.
> The rest of the skill is language-agnostic; only Step 4 needs to be edited.

---

## Step 5 — Render Markdown

Write the digest to `{OUTPUT_DIR}/{week_label}.md`. Template:

```markdown
---
tags: [journal-digest]
date: {date_to}
week: {week_label}
---

# Weekly Journal Digest — {week_label}

> Source: PubMed (edat filter, {date_from} to {date_to}). Journals: Clin Infect Dis, Emerg Infect Dis, MMWR, Transpl Infect Dis, Blood, N Engl J Med. {n} articles total.

## Clin Infect Dis

### {Title}
**Authors**: {Authors} | **Type**: {Article Type} | **Date**: {PubDate}
**DOI**: [{DOI}](https://doi.org/{DOI})
**TL;DR**: 

> **Hot Take**:

---

## Emerg Infect Dis
... (same per-article format)

## MMWR Morb Mortal Wkly Rep
...

## Transpl Infect Dis
...

## Blood
...

## N Engl J Med
...

*Source: PubMed (retrieved {today})*
```

**Formatting rules**:

- Authors: one author → full name; multiple → `{first_author} et al.`
- No DOI → `PMID: {pmid}` instead of the DOI line
- Section with no surviving articles → `> No matching articles this week.`
- Articles separated by `---`

If you swap journals, also rename the `## {section}` headers to match.

---

## Step 6 — Save (optional commit)

The Markdown file lives at `{OUTPUT_DIR}/{week_label}.md`. Stop here unless the
user explicitly asks to commit/push — this fork intentionally does not assume
a publish target.

If the user wants the file committed to their vault:

```sh
git add {OUTPUT_DIR}/{week_label}.md
git commit -m "weekly journal digest {week_label}"
```

If they want to push to a public site repo, that's their choice — add the
relevant `git remote` / `gh pr` / API call as a follow-up step.

---

## Error Handling

| Situation                      | Action                                       |
|--------------------------------|----------------------------------------------|
| Step 1 search fails entirely   | Report and stop                              |
| One journal returns 0 articles | Continue; that section shows a placeholder   |
| CrossRef refill fails          | Continue; affected articles have blank TL;DR |
| Output file already exists     | Overwrite (the week label disambiguates)     |
