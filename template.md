<!--
  Weekly Journal Digest — output template
  Placeholders filled at runtime:
    {week_label}            e.g. 2026-W23
    {date_from}, {date_to}  edat window (YYYY/MM/DD)
    {today}                 retrieval date
    {journal_list}          comma-separated journal names (filled from your Step 1 list)
    {n}                     total article count

  Journal sections: Claude expands the stubs below — one `## Abbreviation` block per
  journal in your configured list. Edit this file to change the frontmatter, reorder
  sections, or rename the annotation label ("Hot Take").
-->
---
tags: [journal-digest]
date: {date_to}
week: {week_label}
---

# Weekly Journal Digest — {week_label}

> Source: PubMed (edat filter, {date_from} to {date_to}). Journals: {journal_list}. {n} articles total.

<!-- more -->

## {Journal 1}

### {Title}
**Authors**: {Authors} | **Type**: {Article Type} | **Date**: {PubDate}
**DOI**: [{DOI}](https://doi.org/{DOI})
**TL;DR**: 

> **Hot Take**:

---

## {Journal 2}
*(same format as above)*

## {Journal 3}
*(same format as above)*

*Source: PubMed (articles retrieved {today})*