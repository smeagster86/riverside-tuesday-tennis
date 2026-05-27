# Documentation Index

**Riverside Tuesday Night Doubles Tennis League Website**

This `docs/` directory + root Markdown files form the complete, version-controlled project memory. The explicit goal (per user requirement): if your computer dies, a Grok builder instance is lost, or you clone onto a completely different machine in 3 months, a single `git clone` + reading 1-2 files gives you *everything* needed to understand, run, extend, or recreate the project.

## Primary Documents (Read in This Order on a Fresh Machine)

1. **Root [README.md](../README.md)**  
   Quick start for players & admin, live site link, high-level "why this repo exists" pitch, and 30-second recover instructions.

2. **[PROJECT_CONTEXT.md](./PROJECT_CONTEXT.md)** (this folder)  
   **The living single source of truth.** Current state, technical architecture, data model with code excerpts, key decisions with rationale & tradeoffs, iteration timeline summary, full artifact inventory (data/, score-sheets/, spreadsheets), detailed recoverability playbook, and open questions. Start here after the root README.

3. **[DECISIONS.md](./DECISIONS.md)** (this folder)  
   Structured, dated log of major architectural, process, and product decisions (lightweight ADR format). Easy to scan and extend. D-009 (May 2026) records the validation, dedup hardening, and rich weekGames persistence work.

4. **Root [LEAGUE_LOGIC_CONSIDERATIONS.md](../LEAGUE_LOGIC_CONSIDERATIONS.md)**  
   Historical deep-dive session notes (2026-05-25). Excellent zoomed-out thinking on league ops edge cases, name matching, court movement, data integrity, analytics, and explicit v1 recommendations. Kept in root as a dated artifact.

## Supporting Artifacts (Also Critical for Recoverability)

- `score-sheets/week-3/` — Raw photos (page1-3.jpeg), master tracker Excel (`RiversideMasterTracker.xlsx`), parsed review spreadsheet, analysis Python scripts (`analyze_week3_parsing.py`, `extract_ground_truth.py`) containing ground-truth transcriptions, parsing challenges, and open questions. These are the "source of truth" materials used to build and validate the admin OCR + review workflow.
- `data/` + `players-initial.json` — Seed JSON for players list and week data.
- `index.html` — The entire runnable application (all logic, UI, data model, OCR pipeline, and many embedded explanations live here).

## How Documentation Evolves

- New major decisions or tradeoffs → Add entry to `DECISIONS.md` + update relevant section of `PROJECT_CONTEXT.md`.
- Significant iteration or refactor → Summarize in `PROJECT_CONTEXT.md` (Iteration History) and reference the commit SHA.
- New league night artifacts (photos, ground truth) → Add to `score-sheets/week-N/` with accompanying notes in the py scripts or a new entry in PROJECT_CONTEXT.
- All docs are plain Markdown → fully searchable, diffable, and readable in any GitHub view or local editor.

**Last updated:** 2026-05 (D-009 hardening + rich per-set persistence for analytics).