# PROJECT_CONTEXT.md — Riverside Tuesday Night Doubles League Website

**Living central document for full project memory and recoverability.**

**Repo:** https://github.com/smeagster86/riverside-tuesday-tennis  
**Live site:** https://riverside-tuesday-tennis.vercel.app/
**Season:** May – September 2026 (22 weeks), Riverside Badminton & Tennis Club (Saskatoon)

> **Purpose of this file:** After cloning on any machine (or opening a fresh builder), read this + the root README + LEAGUE_LOGIC_CONSIDERATIONS.md and you have 95% of the context that existed in the original conversations, iterations, spreadsheets, and photos.

---

## 1. What The Site Does (Current State)

Single-file, phone-first web app for managing a competitive doubles tennis ladder run on paper score sheets.

**Public view (no login):**
- Sortable/filterable standings table with total points, adjusted total (drop 2 lowest weeks), weeks played, and **Recent Form** (normalized mini pips for last 3 weeks).
- Embedded league rules explanation (court groupings of ~4 players, rotate partners, 3 or 5 sets depending on court size, winners move up / lowest move down).
- WhatsApp contact for organizer.

**Admin / Organizer workflow (login: tennisadmin / riverside306!):**
The single, recommended, clean path (as of the May 2026 cleanup):
1. Context questions: # courts set up tonight + # players who actually played (sets N for points).
2. Drop 3–4 WhatsApp photos of handwritten paper score sheets into the large, obvious drop zone in the numbered Organizer portal (#tools section).
3. Tesseract.js runs 100% client-side → produces editable review table (Court | Player | Sets/Games | Court Rank) with live **canonical name suggestions** from master list (fuzzy match via findBestMatch + MASTER_PLAYERS).
4. Organizer corrects OCR errors by tap-edit; names auto-resolve to canonical where possible.
5. Big green "LOOKS GOOD — CALCULATE POINTS & UPDATE STANDINGS" button.
6. Global ranking by games won that night → assign N, N-1, ... 1 points. Updates standings instantly. Persisted to localStorage.

The legacy bulk-paste / manual entry UI and the old duplicate modal commit path were removed in the May 2026 cleanup pass (see D-008 and Architecture notes in index.html). The clean 3-step photo workflow in the main admin portal is now the *only* supported organizer experience.

All processing (OCR, name matching, points, rendering) is client-side for privacy and zero hosting complexity.

**Preloaded:** Accurate Week 3 data (May 20, 2026, 7 courts, 39 players) derived from actual score sheets.

---

## 2. Technical Architecture & Data Model

### Core Stack (deliberately minimal)
- Single `index.html` (Tailwind via CDN + Font Awesome + Chart.js + Tesseract.js v5 via CDN).
- No backend, no database, no auth server. Deployed on Vercel (simple headers via vercel.json + edge middleware for security headers).
- Persistence: browser localStorage only (on the organizer's device).
- Data export: in-app "Download full site" button serializes current seasonData + HTML for offline/backup use.

**Rationale for single-file + client-only:**
- Organizer primarily uses phone (WhatsApp photos arrive there).
- Zero cost, zero maintenance, zero privacy surface (photos never leave device).
- Easy to clone entire runnable state + context in one repo.
- Tradeoff accepted: multi-device sync and long-term backup require manual export or future enhancement (see Open Questions).

### Data Model (in `seasonData` JS object)
```js
{
  currentWeek: 3,
  players: [  // array of { name: string (canonical), points: number[] }  // one entry per week played
    { name: "Chris K.", points: [21, 42, 27] },
    ...
  ],
  weekMeta: [ { num: 1, date: "May 6", n: 28 }, ... ],  // N = actual players who recorded scores that night
  weekGames: {    // raw source of truth (enables corrections + future re-computation)
    "3": [
      { name: "Adrian", gamesWon: 17, court: 1 },
      { name: "Chris K.", gamesWon: 14, court: 1 },
      ...
    ]
  }
}
```

**Key design points captured in code (index.html lines ~648-780, ~1131+, and the clean admin path ~1490+):**
- `getPlayerTotal` / `getAdjustedTotal`: simple sum + drop lowest 2 weeks.
- `computeCurrentStandings`: filters to players with at least one positive score (eliminates sign-up list noise).
- Recent Form pips: for each of last 3 weeks, color intensity = (player gamesWon that week) / (max gamesWon that week) × 100%. This normalizes for wildly different N (28 vs 42).
- Points calculation (global sort by gamesWon in adminCommitReviewedWeek): sort all players who played by gamesWon desc → assign N down to 1 (global, not per-court). Ties broken by name for stability.
- Name handling: `normalizeName()` + `findBestMatch()` (simple contains + starts-with after lowercasing/punct stripping). Used in review table (live suggestions) and on commit (canonical resolution). Master list (`MASTER_PLAYERS`) extracted from RiversideMasterTracker.xlsx (see Artifacts).
- Court grouping: inferred during OCR review from context + editable; stored per entry for future use.

**Ground truth / future-proofing:** The `score-sheets/week-3/extract_ground_truth.py` defines a richer structure with per-court `physical_court_hint`, per-player `set_scores: number[]`, `games_won`, `court_rank`. This was intentionally captured even if the live app currently stores simplified `gamesWon` + `court`. The raw per-set data is prerequisite for planned partner/pairing analytics ("who plays well together?") because you need to know exact sides of the court per set.

### Why Certain Things Are Stored
- Both raw `weekGames` *and* derived `points[]`: allows organizer to correct a past week and re-compute standings without losing history.
- `n` per week in meta: required for correct point scaling and Recent Form normalization.

**Post-cleanup implementation note:** The primary live path (`initAdminPhotoWorkflow`, `renderAdminReviewTable`, `adminCommitReviewedWeek`) lives in the clean numbered portal. The Architecture notes comment block (top of <script>, added during cleanup) is the quickest on-code reference.

---

## 3. Key Decisions & Major Tradeoffs

(See also the dedicated [DECISIONS.md](./DECISIONS.md) for formal lightweight ADR entries. This section gives narrative + cross-refs.)

**D-001: Single-file client-side architecture (Accepted)**  
Chose one self-contained HTML + CDNs over a proper app + backend or Google Sheets sync. Rationale: phone-first organizer, instant privacy (no photos uploaded), trivial recoverability (clone repo = full runnable + docs + data). Tradeoff: localStorage is device-local; multi-device or collaborator use requires export/import for now.

**D-002: In-browser Tesseract OCR as primary score entry path (Accepted)**  
Photos arrive via WhatsApp on the organizer's phone. Running OCR locally (Tesseract.js) eliminates any server or third-party image processing. Review table is fully editable with live fuzzy suggestions. Tradeoff: OCR quality depends on photo clarity + current simple parsing (mixed numeric vs "6-2 6-3" notation handled in review step by human). Analysis scripts in score-sheets/ document the challenges discovered.

**D-003: Global ranking by games won for N-point allocation (Accepted)**  
League rule: everyone who played that night is ranked globally by total games won → top gets N points, next N-1, down to 1. Implemented exactly (see adminCommitReviewedWeek). Not per-court. Rationale: matches existing paper process and social movement (winners up). Court data is still captured for UI grouping and future analytics.

**D-004: Simple fuzzy name canonicalization + manual override instead of full Player ID system (Accepted for v1)**  
`MASTER_PLAYERS` array + `normalizeName`/`findBestMatch` in review UI (index.html:685+). Shows suggestions; commit forces canonical where possible. Rationale: real-world OCR/handwriting produces "Chr1s", "Dewayne E." vs "Dwayne", nicknames. Full ID + merge UI would be heavier for a solo phone tool right now. Documented as high-priority future item in LEAGUE_LOGIC_CONSIDERATIONS.md.

**D-005: Normalized "Recent Form" pips using weekly max (Accepted)**  
Raw points or games won are misleading across weeks because N varies 20–42. Solution: for each of last 3 weeks, show mini pips colored by (player's games / that week's max games). This is the single biggest UX win for comparing performance. Implemented in render logic ~806+.

**D-006: Filter public standings to players with actual recorded scores (Accepted)**  
Prevents noise from people who signed up but never showed. Explicitly called out in LEAGUE file and implemented in computeCurrentStandings (filter + weeksPlayed count).

**D-007: Keep RiversideMasterTracker.xlsx as canonical name source of truth (for now)**  
App hardcodes the list; Excel remains the organizer's trusted master. Future: could become sync target (CSV upload, Sheets API). See LEAGUE section 1.

**D-008: Rich court + per-set ground truth artifacts even if app stores simplified data (Accepted)**  
The py scripts and week3_parsed_review.xlsx capture full set_scores arrays and physical court hints. This was deliberate investment for partner analytics (requires knowing exact pairings per set) and for validating the OCR/parser pipeline against real photos.

---

## 4. Iteration History (High-Level Timeline)

The git history itself is an excellent record (descriptive commit messages). Key phases (remote + local):

- **Early / foundational (multiple "Update index.html" commits):** Basic standings table, rules, initial data model, N-points logic.
- **Major redesign wave:** Full modern UI, interactive sortable standings, powerful local-first score entry implementing exact league rules + preview + export. (See merge commit in history.)
- **Polish for league night (a399eeb, 39e3f64, etc.):** Clean public view (WhatsApp-only for regular players), prominent admin login, preloaded real Week 3, mobile-friendly photo upload + editable review table with context questions.
- **Name recognition & standings hardening (553cbc6):** Added MASTER_PLAYERS, fuzzy matching in admin review, canonical resolution on commit, filter to actual players only. Directly addressed spelling drift and sign-up vs played distinction.
- **Logic considerations doc (01c53e1):** Added LEAGUE_LOGIC_CONSIDERATIONS.md after zoomed-out review of missing edge cases (name identity, court movement, versioning, tiebreakers, backup strategy, partner analytics).
- **Major UX overhaul (66994dc):** 4-step numbered admin portal (Context → Upload → Review nice table → Commit). Big obvious drop zone, clean grouped review table modeled after target format, prominent commit button.
- **May 2026 cleanup pass (working copy edits reflected in subsequent remote index.html fix):** Removed large block of dead legacy code — the entire old bulk-paste/manual entry system (applyBulkPaste, addPlayerToWeekEntry, renderWeekEntryList, calculatePointsFromGames, previewNewStandings, commitWeekToStandings + all related DOM lookups for non-existent elements), the duplicate old commit path `commitReviewedWeek`, and `initScoreEntry()` + its call in initEverything. Added clear "Architecture notes" comment block at the top of the <script> section explaining the current clean state, what was cleaned, the intended single primary admin photo workflow (the numbered Organizer portal), and a pointer to the docs/ folder. Goal: reduce hodgepodge and make the single-file app more cohesive and robust while keeping all real functionality intact. See D-008 for full ADR.
- **Centralize for recoverability (e882aa0 + docs commits 202a778 / 5fea1c2):** Added structured docs/ folder (PROJECT_CONTEXT, DECISIONS, INDEX), updated README, full artifact emphasis. The cleanup and docs work together to make the project trivially understandable on a fresh clone.

**Lesson from iterations:** The biggest value came from repeatedly asking "what would future me need if this machine is gone tomorrow?" and then adding the raw artifacts + explanatory docs + (in this pass) cleaning up the code itself.

Remote tip at time of initial docs creation: 67b9b16a (and nearby). Local working clone + recent remote fixes contain the post-cleanup state.

---

## 5. Data, Artifacts & Ground Truth Inventory

**Critical for "new machine" scenarios:**

- `score-sheets/week-3/` (the heart):
  - page1.jpeg, page2.jpeg, page3.jpeg: Actual handwritten sheets (7 courts, mixed notation: plain numbers vs "6-2 6-3 6-1", sets to 7 on one court).
  - RiversideMasterTracker.xlsx: Master player list + historical scores. Source of the 48-name MASTER_PLAYERS array.
  - Actual Score.png + week3_parsed_review.xlsx: Clean target table format + detailed parsing review with tabs for logic notes, ground truth, analysis.
  - analyze_week3_parsing.py & extract_ground_truth.py: Contain the best-effort manual transcription, recommended parsing pipeline, observed challenges (mixed scoring notation, court group detection via lines + rank resets, IN column for actual attendance, T column as authoritative games won), open questions for Adrian, and a structured ground_truth JSON shape with physical_court_hint + per-player set_scores arrays.

- `data/players.json`, `data/index.json`, `data/weeks/week-*.json`, `players-initial.json`: Current seed data and simplified week exports. Useful for testing or resetting the app.

- `index.html` itself embeds full Week 3 rich entry data (courts 1-7 with specific names/games) for immediate usability after clone.

**How to use on a fresh clone:** Open index.html directly. For re-validation or parser experiments: run the Python scripts against the JPEGs (requires openpyxl + manual photo inspection). The xlsx files are the human source of truth for names.

**Note on binaries in Git:** Photos and xlsx are committed in local history for recoverability. They are large-ish for Git; see Recoverability Playbook below for Git LFS recommendation.

---

## 6. Recoverability Playbook (The Most Important Section)

**Goal:**  `git clone https://github.com/smeagster86/riverside-tuesday-tennis.git`  → open `index.html` in browser  → full working app + full historical context + raw source material.

**Current 30-second steps (as of docs creation + cleanup):**
1. Clone the repo.
2. Open `index.html` (Week 3 data is preloaded; the clean admin photo workflow in the Organizer portal works immediately after admin login).
3. Read root README + this file + LEAGUE_LOGIC_CONSIDERATIONS.md + the Architecture notes comment inside index.html.
4. For full Week 3 re-analysis: open the score-sheets/ files in Excel or run the .py scripts.

**Known Current Gap (Critical):** At the time these docs were added, the full `score-sheets/week-3/` (JPEG photos, xlsx files) and `data/` + `players-initial.json` exist in the local clone's commit history (see local chore commit e882aa0) but have **not yet been pushed to the GitHub remote** in some snapshots. The remote tip contains code + docs/. The README describes the artifacts as present for recoverability. Resolution via Git LFS is still recommended (see earlier commits).

**Resolution path (do this soon):** 
- Install Git LFS: `git lfs install`
- Track the asset paths: `git lfs track "score-sheets/**" "data/**" "*.xlsx" "*.jpeg" "*.png"`
- Commit the .gitattributes, then push the asset-containing commits (or cherry-pick / rebase as needed).
- This keeps the repo size reasonable while guaranteeing the photos and spreadsheets travel with every clone.

**Additional backup strategies already in the app:**
- Inside Admin tools: "Download full site" button produces a self-contained HTML snapshot with current data.
- Organizer should periodically export the master Excel and commit updates to the repo.
- localStorage on phone is the live working copy; treat the repo as the durable source.

**Multi-machine / phone → laptop handoff:** Currently manual (export JSON or full HTML snapshot, load on other device). Future work tracked in Open Questions.

**If everything is lost except this repo:** The combination of photos + ground-truth py + master xlsx + the logic in index.html + this document + the Architecture notes lets you rebuild the entire standings and the admin tool from scratch.

---

## 7. Open Questions & Future Roadmap (Synthesized)

Pulled primarily from LEAGUE_LOGIC_CONSIDERATIONS.md (2026-05-25) and the Week 3 analysis scripts. Many remain valid.

**High priority (near-term):**
- Stronger name matching (top-3 fuzzy suggestions with confidence; Levenshtein + phonetic; eventual stable Player IDs + merge UI).
- Record Physical Court numbers + basic movement history in review/commit.
- Easy rich export (full season JSON/CSV with raw per-set data) from admin tools.
- Validation warnings before committing a week.

**Data integrity & ops:**
- Versioning/audit log for corrections.
- Mid-season player additions & handicaps.
- Tiebreakers for equal final totals.
- Handling byes / very low attendance weeks.
- Per-court vs global point nuances if format changes (sets to 7 on some courts).

**Analytics (the exciting long-term stuff):**
- Partner/pairing performance (requires the rich set_scores + court side data already captured in ground truth).
- Individual trend lines beyond pips.
- Court-by-court dominance.

**Technical / workflow:**
- Multi-device sync for the organizer (or lightweight Google Sheets backend while keeping Excel as heart).
- Optional public photo submit for players (with moderation) so organizer doesn't have to be the only one photographing sheets.
- PWA enhancements for better phone experience.

**Process note:** Every time a new league night happens, add the new photos + any new parsing notes to a `score-sheets/week-N/` folder and update this document + DECISIONS.md if any logic changed.

---

## 8. Documentation Philosophy & Maintenance

We chose a small number of high-signal Markdown files over a wiki or external tool because:
- Everything lives in the same Git repo as the code and raw artifacts.
- Plain text = trivial to diff, search, read offline, and recover.
- One living PROJECT_CONTEXT + one DECISIONS log scales well for a solo project without becoming maintenance burden.
- Git history + these files together give both the "what changed" and the "why we chose it at the time."

When in doubt while working: ask "Will future me on a different computer in 3 months understand the decision and have the raw materials?" Then document it here or in a new DECISIONS entry.

**This file is intended to be updated in place as the single source of living context.**

---

## 9. Post-May-2026-Cleanup State & Remaining Technical Debt

The May 2026 cleanup (D-008) successfully removed the bulk of the legacy hodgepodge. The code + docs now present a clear, cohesive picture:
- The *only* supported admin path is the numbered photo workflow in the Organizer portal (detailed in section 1 and the Architecture notes comment in index.html).
- All real functionality is preserved and the single-file app is easier to understand.

However, the following items were noted during review of the current state (local working tree + GitHub main) but not fully excised or corrected in the initial cleanup pass. They are low-risk to current operation (Week 3 preloads + new photo commits produce correct public standings) but should be addressed for full alignment with the documented architecture and data model:

1. **Dead legacy modal code still present:** The entire `showSubmitPhotoModal()` function (~200 lines, lines ~1120–1390 range) including its duplicate OCR wiring, review list, and an internal `applyBtn.onclick` calling the removed `commitReviewedWeek`. The helper `parseScoreSheetText` is used only by this dead path. Never invoked (handleSubmitScoresClick now correctly scrolls to #tools for admins), but bloats the file.

2. **Broken / stale references in active code:**
   - `initEverything()` contains direct `document.getElementById('entry-week-num').value` and `getElementById('entry-week-label')` (elements do not exist in the current clean admin HTML). These will throw if execution reaches them.
   - `window.RIVERSIDE_LEAGUE` global export references `commitWeekToStandings` (undefined since removal of legacy commit paths).

3. **Incomplete implementation of rich data model in the primary commit path:** `adminCommitReviewedWeek` correctly computes and stores `points[]` using the league rules and canonical names, updates currentWeek, and triggers re-renders. However, it does **not** write entries to `seasonData.weekGames[weekNum]` (array of {name, gamesWon, court}) or append a `{num, date, n}` object to `weekMeta`. Only `preloadWeek3Data` (for the initial Week 3 seed) populates these structures. This means the "raw source of truth" described in section 2 and earlier decisions is aspirational for weeks committed after Week 3. Future analytics (partner performance, re-computation) will require this data.

4. **Minor drift between Architecture notes and live commit function:** The comment accurately describes intent and the clean UI; the implementation of the commit step is leaner than the removed legacy paths but omits the weekGames persistence that the comment and data model section still reference.

5. **Pre-existing (unchanged by cleanup):** Device-local localStorage only; asset recoverability on GitHub still benefits from Git LFS for score-sheets/ and data/ binaries (recommended in section 6).

**Recommended next action:** A short dedicated follow-up cleanup + hardening pass (tracked as D-009) to delete the dead modal block, repair the init references, and extend `adminCommitReviewedWeek` (and any future week commit helpers) to also persist the full weekGames + weekMeta records using the data already present in the review table (`currentReviewData` or equivalent). This will make the live app match the documented rich data model exactly.

These notes ensure that anyone cloning the repo after the cleanup immediately sees both the improved state *and* the precise remaining items, without having to discover them by reading code.

---

*Generated as part of recoverability hardening — May 2026. Updated with cleanup details post D-008.*
