<!--
  Weekly Journal Digest — output template
  Placeholders (filled at runtime):
    {week_label}            e.g. 2026-W23
    {date_from}, {date_to}  edat window (YYYY/MM/DD)
    {today}                 retrieval date
    {nejm_id_issue_date}    NEJM ID monthly issue date
    {nejm_heme_issue_date}  NEJM Hema/Onc monthly issue date
    {prior_week}            previous week label (used when newsletter already ingested)

  To customise: edit frontmatter tags/categories, reorder or remove journal sections,
  adjust the article-block fields, or change the footer lines.
  Behavioural rules (DOI vs tracking-URL, pathogen italics, etc.) live in SKILL_cloud.md / SKILL_local.md.
-->
---
tags: [journal-digest]
date: {date_to}
week: {week_label}
---

# Weekly Journal Digest — {week_label}

> 資料來源：PubMed（edat 過濾，{date_from} 至 {date_to}）。涵蓋期刊：Clin Infect Dis、Emerg Infect Dis、MMWR、N Engl J Med（Originals + Reviews，無主題過濾）、Transpl Infect Dis、Blood。共 __ 篇。NEJM Cases / Images / Perspective / Correspondence 由本月 NEJM ID 與 Hema/Onc 月報（Gmail）補完。

<!-- more -->

## Clin Infect Dis

### {Title}
**Authors**: {Authors} | **Type**: {Article Type} | **Date**: {PubDate}
**DOI**: [{DOI}](https://doi.org/{DOI})
**TL;DR**: 

> **嘻嘻/不嘻嘻**：

---

## Emerg Infect Dis
...（同上格式）

## MMWR Morb Mortal Wkly Rep
...

## Transpl Infect Dis
...

## Blood
...（同上格式）

## N Engl J Med
...（同上格式）

## NEJM Monthly — Infectious Disease（{nejm_id_issue_date}）

> 資料來源：Gmail（Infectious Disease Update from NEJM，發行日 {nejm_id_issue_date}）。補完 PubMed N Engl J Med section 之外的 Cases / Images / Perspective / Correspondence 等，無主題過濾。

### {Title}
**Authors**: {Authors} | **Type**: {Article Type} | **Date**: {nejm_id_issue_date}
**DOI**: [{DOI}](https://doi.org/{DOI})
**TL;DR**: 

> **嘻嘻/不嘻嘻**：

---

## NEJM Monthly — Hematology/Oncology（{nejm_heme_issue_date}）

> 資料來源：Gmail（Hematology/Oncology Update from NEJM，發行日 {nejm_heme_issue_date}）。補完 PubMed N Engl J Med section 之外的 Cases / Images / Perspective / Correspondence 等。已濾除標題含 cancer / carcinoma 之篇目。

### {Title}
**Authors**: {Authors} | **Type**: {Article Type} | **Date**: {nejm_heme_issue_date}
**DOI**: [{DOI}](https://doi.org/{DOI})
**TL;DR**: 

> **嘻嘻/不嘻嘻**：

*資料來源：PubMed（articles retrieved {today}）*
*NEJM 月報資料來源：Gmail；abstract / DOI 由 PubMed 標題搜尋（fallback CrossRef）回查取得；newsletter body 不含 abstract，追蹤 URL 不可解析。*