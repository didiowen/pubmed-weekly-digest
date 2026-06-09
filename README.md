# Pubmed Weekly Digest

> [繁體中文版](./README.zh-TW.md)

A Claude Code skill that builds a weekly digest of articles published in the last 7 days across a configurable list of PubMed-indexed journals, annotates each with a TL;DR and a one-line **Hot Take** (positive or snarky), and writes the result to a local Markdown file. Output is in English by default and the language is one prompt-edit away from anything else.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill%20Based-blueviolet?logo=anthropic)](https://claude.ai/claude-code)
[![Made in Taiwan](https://img.shields.io/badge/Made%20in-Taiwan%20%F0%9F%87%B9%F0%9F%87%BC-red)](https://github.com/htlin222/society-calendar)

## Folder layout

```
pubmed-weekly-digest/
├── SKILL.md          # quickstart: simple, English, PubMed only
├── SKILL_cloud.md    # full-featured: Chinese TL;DR + comment, Gmail newsletter, PMID dedup
├── SKILL_local.md    # local-only: same as cloud + redirect DOI resolution + institutional browser
├── README.md         # this file
├── README.zh-TW.md   # Chinese version
└── LICENSE
```

No bundled scripts — the entire skill is prompt-driven.

## Which skill file to use?

| File | When to use | Requirements |
|------|-------------|--------------|
| `SKILL.md` | Quickstart — English output, any journal list, no Gmail needed | PubMed MCP |
| `SKILL_cloud.md` | Full-featured — Traditional Chinese TL;DR + 嘻嘻/不嘻嘻 comment, NEJM Gmail newsletter (ID + Hema/Onc monthly), PMID dedup, OA abstract fallback; works on cloud and local | PubMed MCP + Gmail MCP |
| `SKILL_local.md` | Same as `SKILL_cloud.md`, plus redirect-based DOI resolution for NEJM newsletter tracking links and institutional browser fallback for paywalled abstracts | PubMed MCP + Gmail MCP + local Claude-in-Chrome |

All three share the same journal list, ISO week-label logic, and CrossRef abstract refill. The full-featured variants add a Gmail newsletter layer (NEJM Infectious Disease + Hematology/Oncology monthly updates) and a smarter abstract pipeline.

> `SKILL.md` is intentionally kept as the simple, language-agnostic entry point. For a production setup with all the features, start with `SKILL_cloud.md` instead.

## Dependencies

- **Claude Code** (with the Skill system) — `https://claude.com/claude-code`
- **PubMed MCP server** configured in your Claude Code MCP settings
- **Gmail MCP server** *(only for `SKILL_cloud.md` / `SKILL_local.md`)* — needed to pull the NEJM monthly newsletters. If you subscribed to NEJM newsletters with a Gmail account, the Gmail MCP works out of the box. If you used a different address (e.g. ProtonMail), set up forwarding to Gmail first.

## Setup

1. Drop this folder into the skills directory of any project: `<project>/.claude/skills/pubmed-weekly-digest/`.
2. Confirm the PubMed MCP server is configured (and Gmail MCP if using the full-featured variants).
3. Open the skill file you chose and edit the **Configuration** block at the top:
   - `OUTPUT_DIR`: where to write the digest (default `output/` inside this skill folder)
   - `JOURNALS`: the list in Step 1 (only in `SKILL.md`)
   - `CROSSREF_MAILTO`: any valid email — used as the CrossRef polite-pool identifier
   - `SKILL_DIR`: path to this skill folder, used for the PMID dedup cache (`SKILL_cloud.md` / `SKILL_local.md` only)
4. Trigger from Claude Code: ask it to "run the weekly journal digest" or "跑週報", or invoke the skill by name.

## Customising the journal list

This is the main thing you'll want to change.

### Where the list lives

`SKILL.md` → **Step 1 — PubMed search** has a table of journal queries. Each row becomes one parallel `mcp__PubMed__search_articles` call. Replace rows to follow different journals, add rows for more journals, or delete rows you don't care about. **Also update the section list in Step 5** — each journal gets its own `## {Journal name}` block in the Markdown template.

### Finding the right journal abbreviation

PubMed's `[Journal]` field expects the **NLM Title Abbreviation**, not the full name. Three ways to find it:

1. Search the journal on PubMed → open any recent article → the citation line shows the abbreviation, e.g. `Clin Infect Dis. 2026 Apr 30;82(8):...`.
2. Browse https://www.ncbi.nlm.nih.gov/nlmcatalog/journals → look up by full title → the "NLM Title Abbreviation" field is what to paste into `[Journal]`.
3. Use `mcp__PubMed__search_articles` once with the full name in your test query — PubMed normalises it and the response shows the abbreviation it actually used.

Common abbreviations:

| Full title | `[Journal]` value |
|---|---|
| Clinical Infectious Diseases | `Clin Infect Dis` |
| Emerging Infectious Diseases | `Emerg Infect Dis` |
| The New England Journal of Medicine | `N Engl J Med` |
| The Lancet | `Lancet` |
| The Lancet Infectious Diseases | `Lancet Infect Dis` |
| JAMA | `JAMA` |
| Journal of Clinical Oncology | `J Clin Oncol` |
| Nature Medicine | `Nat Med` |
| The Journal of Infectious Diseases | `J Infect Dis` |
| Blood | `Blood` |
| BMJ | `BMJ` |
| Annals of Internal Medicine | `Ann Intern Med` |

### Picking `max_results`

The MCP wrapper caps at **200 per query**. Rough sizing for a 7-day window:

- High-volume specialty journals (CID, EID, Blood): `50` is usually plenty.
- Weekly bulletins (MMWR, Lancet): `30` covers one issue.
- High-impact generals (NEJM, JAMA): `50` is a safe upper bound; one issue is usually under 30.
- Monthly journals (most subspecialty journals): `30` is more than enough.

### Two PubMed MCP pitfalls to remember

- **Wildcards `*` are not supported.** Use `mycobacterium` instead of `mycobacteri*`, or expand explicitly: `mycobacterium OR tuberculosis OR NTM`.
- **The boolean-operator cap is 20.** A single query with more than 20 `OR`/`AND` operators fails with `INVALID_PARAMETERS`. Split into two parallel queries if you hit this.

## Adapting the output style

The annotations in Step 4 are written in **English by default**, with the one-line comment framed as a **Hot Take**. Both choices are pure prompting — adapt as you like:

- **Language**: edit the instructions in Step 4 to specify your preferred language (e.g. Traditional Chinese, Spanish, Japanese). Claude translates abstracts on the fly; no other part of the skill needs changing.
- **Tone label**: rename "Hot Take" to whatever framing fits — "key takeaway", "clinical implication", "bottom line", etc.
- **Length / depth**: the default ask is 1–2 sentences. Loosen it to a paragraph if you want more detail.

## License

MIT — see [`LICENSE`](./LICENSE).