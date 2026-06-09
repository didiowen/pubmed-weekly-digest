# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [2.0.0] - 2026-06-09

### Added

- **`SKILL_cloud.md`** — Full-featured variant. Adds on top of `SKILL.md`:
  - Subagent B (Gmail MCP): pulls the NEJM Infectious Disease and Hematology/Oncology monthly newsletters, parses each article, and back-fills DOI + abstract via PubMed title search and CrossRef fallback.
  - NEJM newsletter gate: detects whether the current month's newsletter has already been ingested in a prior weekly digest, and skips re-fetching to avoid duplicates.
  - Cross-week PMID dedup: tracks seen PMIDs in `.seen_pmids.json` with a 14-day rolling window to suppress duplicate entries from epub-vs-print double-indexing or overlapping search windows on a retry run.
  - OA abstract fallback (Step C2.5): queries Unpaywall and Europe PMC for open-access full text before falling back to a "no abstract" annotation.
  - Chinese-language annotations: Traditional Chinese TL;DR (60–90 characters) with pathogen italicisation rules, plus a 嘻嘻/不嘻嘻 one-liner (positive or snarky, ≤40 characters).
  - Works on both cloud and local Claude Code environments.

- **`SKILL_local.md`** — Local-environment extension of `SKILL_cloud.md`. Adds:
  - Step B3.0: follows `t.n.nejm.org` tracking-URL redirects to `www.nejm.org/doi/…` and extracts the DOI directly, bypassing the multi-stage PubMed title-search fallback for newsletter articles. Cloud TLS proxies block this cross-domain redirect; the step is kept local-only.
  - Step C2.5 extension: after the OA check, tries Claude-in-Chrome with an institutional browser session to retrieve paywalled full text. Skips cleanly if no browser session is available or bot-detection blocks the request.
  - Newsletter articles where the DOI was resolved via B3.0 display a standard `**DOI**:` line instead of a newsletter tracking link, matching the format of PubMed-sourced articles.

- **`README.md`** / **`README.zh-TW.md`**: added a skill-variant comparison table, a Gmail MCP setup note (direct Gmail subscription vs. forwarding), and updated the folder layout and setup steps.

### Notes

- `SKILL.md` is **unchanged**. It remains the simple, language-agnostic, Gmail-free quickstart.
- The full-featured variants hardcode a six-journal list (CID, EID, MMWR, Transpl Infect Dis, Blood, N Engl J Med) tailored for ID + haematology. Swap the Subagent A journal table and Step 5 section headers for other specialties.
- Gmail MCP is required for the newsletter layer. If your NEJM subscription uses a Gmail address the MCP works out of the box; otherwise forward the newsletters to Gmail first.

## [1.0.0] - 2026-05-18

### Added

- `SKILL.md`: PubMed edat search across a configurable journal list, CrossRef abstract refill, English TL;DR + Hot Take annotation, Markdown digest output. No Gmail dependency. Language and journal list are one-prompt-edit away from anything else.
- `README.md` / `README.zh-TW.md`: setup guide, journal-abbreviation reference, `max_results` sizing notes, and topic-filter examples.