# Pubmed Weekly Digest

> A Claude Code skill that builds a weekly digest of articles published in the last 7 days across a configurable list of PubMed-indexed journals, annotates each with a TL;DR and a one-line **Hot Take** (positive or snarky), and writes the result to a local Markdown file. Output is in English by default and the language is one prompt-edit away from anything else.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill%20Based-blueviolet?logo=anthropic)](https://claude.ai/claude-code)
[![Made in Taiwan](https://img.shields.io/badge/Made%20in-Taiwan%20%F0%9F%87%B9%F0%9F%87%BC-red)](https://github.com/htlin222/society-calendar)

## Folder layout

```
pubmed-weekly-digest/
├── SKILL.md     # the skill
├── README.md    # this file
└── LICENSE
```

No bundled scripts — the entire skill is prompt-driven.

## Dependencies

- **Claude Code** (with the Skill system) — `https://claude.com/claude-code`
- **PubMed MCP server** configured in your Claude Code MCP settings
- That's it. No Python deps, no `.env` required for a minimal run.

## Setup

1. Drop this folder into the skills directory of any project: `<project>/.claude/skills/pubmed-weekly-digest/`.
2. Confirm the PubMed MCP server is configured.
3. Open `SKILL.md` and edit the **Configuration** block at the top:
   - `OUTPUT_DIR`: where to write the digest (default `output/` inside this skill folder)
   - `JOURNALS`: the list in Step 1
   - `CROSSREF_MAILTO`: any valid email — used as the CrossRef polite-pool identifier
4. Trigger from Claude Code: ask it to "run the weekly journal digest" or "跑週報", or invoke the skill by name.

## Customising the journal list

This is the main thing you'll want to change.

### Where the list lives

`SKILL.md` → **Step 1 — PubMed search** has a table:

```markdown
| Query string                          | max_results |
|---------------------------------------|-------------|
| `Clin Infect Dis[Journal]`            | 50          |
| `Emerg Infect Dis[Journal]`           | 50          |
| `MMWR Morb Mortal Wkly Rep[Journal]`  | 30          |
| `Transpl Infect Dis[Journal]`         | 50          |
| `Blood[Journal]`                      | 50          |
| `N Engl J Med[Journal]`               | 50          |
```

Each row becomes one parallel `mcp__PubMed__search_articles` call. Replace rows to follow different journals, add rows for more journals, or delete rows you don't care about. **Also update the section list in Step 5** — each journal gets its own `## {Journal name}` block in the Markdown template.

### Finding the right journal abbreviation

PubMed's `[Journal]` field expects the **NLM Title Abbreviation**, not the full name. Three ways to find it:

1. Search the journal on PubMed → open any recent article → the citation line shows the abbreviation, e.g. `Clin Infect Dis. 2026 Apr 30;82(8):...`.
2. Browse https://www.ncbi.nlm.nih.gov/nlmcatalog/journals → look up by full title → the "NLM Title Abbreviation" field is what to paste into `[Journal]`.
3. Use `mcp__PubMed__search_articles` once with the full name in your test query — PubMed normalises it and the response shows the abbreviation it actually used.

Common abbreviations:

| Full title                              | `[Journal]` value         |
|-----------------------------------------|---------------------------|
| Clinical Infectious Diseases            | `Clin Infect Dis`         |
| Emerging Infectious Diseases            | `Emerg Infect Dis`        |
| The New England Journal of Medicine     | `N Engl J Med`            |
| The Lancet                              | `Lancet`                  |
| The Lancet Infectious Diseases          | `Lancet Infect Dis`       |
| JAMA                                    | `JAMA`                    |
| Journal of Clinical Oncology            | `J Clin Oncol`            |
| Nature Medicine                         | `Nat Med`                 |
| The Journal of Infectious Diseases      | `J Infect Dis`            |
| Blood                                   | `Blood`                   |
| BMJ                                     | `BMJ`                     |
| Annals of Internal Medicine             | `Ann Intern Med`          |

### Picking `max_results`

The MCP wrapper caps at **200 per query**. Rough sizing for a 7-day window:

- High-volume specialty journals (CID, EID, Blood): `50` is usually plenty.
- Weekly bulletins (MMWR, Lancet): `30` covers one issue.
- High-impact generals (NEJM, JAMA): `50` is a safe upper bound; one issue is usually under 30.
- Monthly journals (most subspecialty journals): `30` is more than enough.

If you regularly see results truncated, raise the number. Don't blindly use 200 — the metadata-fetch step fans out in batches of 20, so larger result sets get slow.

### Adding topic filters to a high-volume journal

For generals like NEJM or JAMA you may want to keep only articles touching specific topics. Use boolean AND with `[tiab]` (title/abstract):

```
N Engl J Med[Journal] AND (infection[tiab] OR antimicrobial[tiab] OR sepsis[tiab] OR HIV[tiab] OR transplant[tiab])
```

This narrows to ID-relevant NEJM articles while still using `edat` for the time window.

### Two PubMed MCP pitfalls to remember

These are real, learned by trial:

- **Wildcards `*` are not supported.** Use `mycobacterium` instead of `mycobacteri*`, or expand explicitly: `mycobacterium OR tuberculosis OR NTM`.
- **The boolean-operator cap is 20.** A single query with more than 20 `OR`/`AND` operators fails with `INVALID_PARAMETERS`. Split into two parallel queries if you hit this — the rest of the skill happily merges the results.

### Testing a query before committing it to SKILL.md

Try the query on the PubMed web UI first:

```
N Engl J Med[Journal] AND (infection OR sepsis) AND 2026/05/04:2026/05/10[edat]
```

If the web search returns what you expect, the same query through the MCP (minus the date range, which you pass as `date_from`/`date_to`) will too.

## Adapting the output style

The annotations in Step 4 are written in **English by default**, with the one-line comment framed as a **Hot Take** that can swing positive or snarky depending on the content. Both choices are pure prompting — adapt as you like:

- **Language**: edit the instructions in Step 4 to specify your preferred language (e.g. Traditional Chinese, Spanish, Japanese, French). PubMed and CrossRef return abstracts in their original language — usually English — and Claude translates on the fly during annotation. No other part of the skill needs changing.
- **Tone label**: rename "Hot Take" to whatever framing fits — "key takeaway", "clinical implication", "bottom line", etc. The two-line per-article block in Step 5 (TL;DR + quote) stays the same; only the words change.
- **Length / depth**: the default ask is 1–2 sentences. Loosen it to a paragraph if you want more detail, or tighten it to one sentence for skim-friendliness.

## License

MIT — see [`LICENSE`](./LICENSE).
