# DECISIONS.md — Architecture & Process Decision Log

**Lightweight ADR (Architecture Decision Record) format for the Riverside Tuesday Night Doubles League site.**

Each entry captures a non-trivial choice, the context at the time, alternatives, the decision, rationale, tradeoffs, and current status. This makes it trivial for future-you (or collaborators) on a new machine to understand *why* things are the way they are without having to reverse-engineer code or old chat logs.

Format template for new entries:

```
## D-XXX: Short Descriptive Title

**Date:** YYYY-MM-DD  
**Status:** Accepted / Implemented / Deprecated / Superseded by D-YYY  
**Deciders:** Adrian + Grok (or whoever)

**Context / Problem:** ...

**Options Considered:**
- Option A ...
- Option B ...

**Decision:** ...

**Rationale:** ...

**Trade-offs & Consequences:** ...

**Artifacts / Implementation:** Links to code lines, docs, commits.
```

---

## D-001: Single-File Client-Side Architecture (No Backend)

**Date:** ~2026-05 (multiple early iterations)  
**Status:** Accepted / Implemented

**Context / Problem:** League is run by one organizer on a phone using paper sheets photographed and sent via WhatsApp. Need a tool that is instantly usable, private, zero-cost to host/maintain, and fully recoverable by cloning one repo.

**Options Considered:**
- Traditional web app (frontend + Node/Sheets backend)
- Google Sheets as primary UI + custom frontend
- Desktop spreadsheet + manual web publishing
- Single self-contained HTML file

**Decision:** Single `index.html` using CDNs for Tailwind, Tesseract, Chart.js. All logic, UI, data model, OCR, and calculations in one file. Deployed statically on Vercel only for convenience.

**Rationale:** Clone = entire runnable app + all planning docs + raw photos + spreadsheets. Zero privacy surface (photos stay on device). Zero ongoing cost or ops. Perfect match for "recover everything on a new machine in one step."

**Trade-offs & Consequences:**
- Excellent for solo phone-first use and recoverability.
- localStorage is per-device → multi-device handoff or backup requires explicit export (the app provides a "Download full site" button).
- No easy multi-organizer collaboration yet.

**Artifacts / Implementation:** index.html (entire app), vercel.json, middleware.js, root README emphasis on recoverability, this PROJECT_CONTEXT.md.

---

## D-002: In-Browser Tesseract OCR for Photo-Based Score Entry

**Date:** 2026-05 (major UX iterations)
**Status:** Accepted / Implemented

**Context / Problem:** Organizer receives 3–4 photos of handwritten sheets on phone. Transcription must be fast, accurate enough with human review, and must never send private score photos to any server.

**Options Considered:**
- Manual transcription into form every week
- Cloud OCR service (Google Vision, etc.)
- Local Tesseract via WASM / JS (Tesseract.js)

**Decision:** Use Tesseract.js v5 loaded from CDN. Photos dropped → processed entirely in browser → editable review table with canonical name suggestions. Human corrects in place.

**Rationale:** Matches the real workflow (photos on the phone). Strong privacy guarantee. Review step turns OCR imperfections into a strength (organizer is in the loop and sees suggestions).

**Trade-offs & Consequences:**
- OCR is good but not perfect on handwriting/mixed notation ("6 6 2" vs "6-2 6-3 6-1" vs sets-to-7); human review table is essential (documented extensively in score-sheets/week-3/*.py and week3_parsed_review.xlsx).
- No recurring cloud costs or data leakage.

**Artifacts / Implementation:** index.html (drop zone + OCR progress + review table code ~1464+), Tesseract CDN script tag, LEAGUE_LOGIC_CONSIDERATIONS.md section 4, analysis Python scripts.

---

## D-003: Global N-Points Ranking by Games Won (Not Per-Court)

**Date:** Early design + confirmed in name-recognition commit
**Status:** Accepted / Implemented

**Context / Problem:** League has explicit rule: rank everyone who played that night globally by total games won that night, then award N, N-1, ..., 1 points where N = actual attendees with scores. Paper process already does global ranking after courts are recorded.

**Options Considered:**
- Per-court ranking and points (would match physical movement more directly)
- Global ranking (current league practice)

**Decision:** Implement exact global sort by gamesWon desc → points N down to 1. Still capture court for grouping in review UI and future analytics.

**Rationale:** Faithfulness to existing league rules and social dynamic (winners move up). Court data remains available.

**Trade-offs & Consequences:**
- Simple, matches paper.
- Some organizer mental work (per-court then global) is now explicit in the review step.

**Artifacts / Implementation:** calculatePointsFromGames() in index.html, weekGames storage, LEAGUE_LOGIC_CONSIDERATIONS.md section 4 (per-court vs global discussion).

---

## D-004: Fuzzy Name Canonicalization with Master List + Live Suggestions + Manual Override (v1)

**Date:** 2026-05- (commit 553cbc6 and follow-ups)
**Status:** Accepted for v1; planned evolution

**Context / Problem:** Player names drift across weeks and especially in OCR output ("Chris K." / "C K" / "Chr1s", "Dewayne E." vs "Dwayne", nicknames, handwriting). Master source is the organizer's Excel. Need reliable matching without forcing perfect spelling every time.

**Options Considered:**
- Strict exact match only (brittle)
- Simple normalize + contains/starts-with (current)
- Full Levenshtein distance + phonetic (Soundex/Metaphone) + confidence scores + top-3 UI
- Stable internal Player IDs in master + ID-based storage
- Post-commit merge tool for discovered duplicates

**Decision:** Implement `normalizeName()` + `findBestMatch()` (index.html ~685) using the MASTER_PLAYERS list. Show live suggestions in review table. On commit, resolve to canonical name when possible. Keep manual edit always available.

**Rationale:** Good enough for real use tonight/tomorrow. Captures the 80% case with very little code. Full ID system or advanced fuzzy was deferred to keep the phone tool lightweight.

**Trade-offs & Consequences:**
- Solves immediate spelling/OCR pain.
- Still risk of bad OCR producing unmatchable names (human must catch).
- Future stronger matching tracked in LEAGUE file and Open Questions.

**Artifacts / Implementation:** MASTER_PLAYERS constant + findBestMatch (index.html), admin review table code, LEAGUE_LOGIC_CONSIDERATIONS.md section 2, RiversideMasterTracker.xlsx.

---

## D-005: Normalized Recent Form Using Per-Week Max Percentage

**Date:** Core design (visible in early polished versions)
**Status:** Accepted / Implemented

**Context / Problem:** Attendance (N) swings dramatically week to week (28 vs 42). Raw points or games-won trends are misleading or noisy. Players and organizer need a fair way to see "how did this person do relative to the field that night?"

**Options Considered:**
- Raw points or games won in recent columns
- Rank within week
- Percentage of that week's maximum games won (chosen)

**Decision:** For the last 3 weeks, render mini pips whose color intensity = (player gamesWon / maxGamesWonThatWeek). Tooltip explains the normalization.

**Rationale:** Directly solves the variable-attendance problem identified early. One of the highest-leverage UX decisions.

**Trade-offs & Consequences:**
- Excellent comparability across weeks.
- Requires storing (or computing) the per-week max; weekMeta.n helps but actual max from the players present is used.

**Artifacts / Implementation:** Recent Form rendering logic (~806+ in index.html), weekGames data, UI explanation text in standings table.

---

## D-006: Filter Standings to Players With Actual Positive Scores

**Date:** 2026-05 (name recognition hardening)
**Status:** Accepted / Implemented

**Context / Problem:** Sign-up sheets include people who never showed. Showing them in standings pollutes the ladder and makes "weeks played" meaningless.

**Options Considered:**
- Show everyone in the master list (with 0s for missed weeks)
- Show only players who have recorded at least one positive score (chosen)

**Decision:** In computeCurrentStandings(), filter to players where `points.some(score => score > 0)`. Also compute weeksPlayed count.

**Rationale:** "Only players who actually recorded a score should appear in public standings (sign-ups who never showed are noise)." — directly from early requirements and LEAGUE file.

**Trade-offs & Consequences:**
- Cleaner, more accurate public view.
- Organizer still has the full master list for name suggestions and future additions.

**Artifacts / Implementation:** computeCurrentStandings() (index.html ~762), LEAGUE_LOGIC_CONSIDERATIONS.md section 1, standings table render.

---

## D-007: Rich Ground-Truth Artifacts With Per-Set Scores & Physical Court Hints

**Date:** Week 3 analysis (concurrent with site development)
**Status:** Accepted (for artifacts & future use)

**Context / Problem:** Current app stores simplified per-player gamesWon + court for the live tool. However, the paper sheets contain full per-set scores and the physical court numbers have meaning for movement rules and pairing analysis.

**Options Considered:**
- Store only final T (games won) in the app for v1
- Capture full detail (set arrays, physical hints) in analysis artifacts even if app is simplified

**Decision:** Invest in detailed transcription in extract_ground_truth.py + week3_parsed_review.xlsx + the ground truth JSON shape (physical_court_hint, set_scores per player per court). Keep the app focused on gamesWon + court for now.

**Rationale:** The richer data is cheap to capture once (during the photo analysis that was already happening) and is a prerequisite for the partner analytics the organizer wants later. Also invaluable for validating/improving the OCR parser.

**Trade-offs & Consequences:**
- More work during Week 3 setup (but paid off in understanding notation edge cases).
- App data model stays simple; migration path exists via weekGames + future richer week storage.

**Artifacts / Implementation:** score-sheets/week-3/extract_ground_truth.py (full structure), analyze_week3_parsing.py (parsing pipeline + open questions), week3_parsed_review.xlsx, LEAGUE_LOGIC_CONSIDERATIONS.md section 7 (analytics).

---

## D-008: Legacy Bulk-Paste / Manual Entry System Removal and Consolidation to Single Primary Admin Photo Workflow (May 2026 Cleanup Pass)

**Date:** 2026-05-26 (cleanup performed in working copy; reflected in remote after index.html restoration commit)
**Status:** Implemented (with follow-up debt tracked)

**Deciders:** Main agent (cleanup); Documentation Curator (this record)

**Context / Problem:** 
The single-file `index.html` had grown a "hodgepodge" of overlapping admin score-entry mechanisms across iterations:
- The original bulk-paste + manual form system (applyBulkPaste, addPlayerToWeekEntry, renderWeekEntryList, calculatePointsFromGames, previewNewStandings, commitWeekToStandings, and all associated lookups for non-existent elements: #bulk-paste, #player-select, #week-entry-list, #points-preview-area, etc.).
- The old full photo modal flow (showSubmitPhotoModal + the duplicate commitReviewedWeek path + initScoreEntry).
- The newer, cleaner numbered photo upload + editable review table UX (Context → Drop photos → Review table → Commit).

This made the file harder to reason about, increased risk of stale references, and obscured the one intended organizer workflow (phone + WhatsApp photos).

**Options Considered:**
- Leave all paths in place and document around the accretion.
- Full simultaneous removal of every legacy line (risk of breakage during the pass).
- Targeted surgical removal of the dead bulk/manual systems + duplicate commit path + initScoreEntry, plus explicit Architecture notes comment + docs update (chosen).

**Decision:** 
During the May 2026 cleanup pass:
- Removed the entire old bulk-paste/manual entry system and every reference to its non-existent DOM.
- Removed the duplicate `commitReviewedWeek` function and its call site inside the legacy modal.
- Removed `initScoreEntry()` and its call inside initEverything (it only supported the dead UI).
- Added a prominent "Architecture notes" block at the top of the `<script>` section (approx lines 529–543) that explicitly states:
  - The single primary admin workflow is the numbered Organizer portal in #tools (Context questions → large photo drop zone → live-editable review table with fuzzy canonical name suggestions → big green "LOOKS GOOD — CALCULATE POINTS & UPDATE STANDINGS" button powered by adminCommitReviewedWeek + renderAdminReviewTable).
  - All commits should go through the new path or preloadWeek3Data.
  - Legacy paths were removed to reduce hodgepodge.
  - Pointer to the docs/ folder for full history.
- Ensured handleSubmitScoresClick (admin case) scrolls to the clean #tools section.

**Rationale:** 
A single-file app benefits enormously from cohesion. Future-you (or any new-machine clone) should be able to read the comment + docs/PROJECT_CONTEXT.md + DECISIONS.md and immediately know the one supported, robust happy path without wading through dead code. The cleanup directly addressed the "hodgepodge" goal stated in the task while keeping every piece of real, working functionality intact (OCR, name matching via findBestMatch/MASTER_PLAYERS, global N-point allocation, Recent Form pips, localStorage persistence, export, etc.).

**Trade-offs & Consequences:**
- The app is now markedly easier to maintain and understand.
- Because the pass was focused on removal of the *bulk/manual* and *duplicate commit* pieces, a modest amount of dead scaffolding from the old modal path remains in the file (see Remaining Technical Debt section in PROJECT_CONTEXT.md). This is explicitly documented rather than hidden.
- The primary live commit path (adminCommitReviewedWeek) is lean and correct for points/standings but does not yet write the full `seasonData.weekGames` + `weekMeta` structures that the data model and earlier docs describe as the rich source of truth for future analytics and re-computation. (Only Week 3 preload populates them today.)
- No user-visible behavior change for organizers or players.

**Artifacts / Implementation:**
- `index.html`: Architecture notes comment, deletions of legacy functions, new clean workflow code (`initAdminPhotoWorkflow` ~1490, `adminCommitReviewedWeek` ~1570, `renderAdminReviewTable` ~1615, `parseAdminPhotoText`), updated initEverything.
- This D-008 entry.
- Updates to `docs/PROJECT_CONTEXT.md` (Iteration History + new Remaining Technical Debt subsection) and README emphasis on the clean photo workflow.
- docs/INDEX.md (cross-reference).

**Follow-up tracked:** D-009 (or next) – Complete excision of the remaining dead `showSubmitPhotoModal` block + parseScoreSheetText + fix init DOM references + make the commit path fully populate weekGames/weekMeta for the documented data model.

---

## Future Decision Template

When a significant choice arises (new backend, stronger name system, tiebreaker rules, multi-device strategy, etc.), add the next numbered entry here using the template above. Then summarize the outcome in PROJECT_CONTEXT.md.

**Next likely candidates (from current open questions):**
- D-009: Complete remaining dead-code excision after D-008 cleanup + full weekGames population in adminCommitReviewedWeek
- D-010: Stable Player IDs + merge UI for name identity
- D-011: Physical Court tracking + movement history
- D-012: Multi-device / Google Sheets sync strategy

---

*Maintained alongside the code for long-term project memory.*