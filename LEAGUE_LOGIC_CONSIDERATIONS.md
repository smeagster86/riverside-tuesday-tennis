# Riverside Tuesday Doubles - League Management Logic Considerations

**Date:** 2026-05-25 (late night session)
**Context:** Final polish pass before league night. User requested zoomed-out review of potential missing logic for running a tennis/sports ladder league with photo-based score entry.

This document captures high-level considerations that may not yet be implemented in the current single-file `index.html` site. The goal is to have a solid operational tool now, with clear paths for future robustness.

---

## 1. Data Integrity & Master Data Management

### Current State
- Master player list is hardcoded in JS (sourced from your `RiversideMasterTracker.xlsx`).
- Weekly data is entered via photo OCR + manual review, then committed to localStorage.

### Potential Gaps / Future Needs
- **Single Source of Truth**: Long-term, the Excel master tracker should probably remain the canonical source. The website could become a *viewer + light editor* that syncs from/to the spreadsheet (Google Sheets API, CSV upload, or file watcher).
- **Versioning / Audit Log**: When an organizer corrects a past week's scores, there is currently no record of what changed or who changed it. For disputes, this matters.
- **Mid-season player additions**: How do we handle someone who joins after Week 1? Do they start with 0s, or get a handicap?
- **Player "active" status**: Currently we filter to "has at least one positive score". This is good, but we should also track "signed up but never showed" vs "active this season".

**Recommendation for v1 (tonight)**: The name matching + "only show players with actual scores" logic we just added is a strong start. Document that the Excel remains the master for now.

---

## 2. Name Matching & Identity Resolution (High Priority - Partially Addressed)

### Current State (as of latest commit)
- `normalizeName()` + `findBestMatch()` in the admin review table.
- Suggestions shown during photo review.
- Commit resolves to canonical name from master list when possible.

### Remaining Risks
- OCR can produce very bad names ("Chr1s K" , "Adr1an", "Dewayne E." vs "Dewayne").
- Different weeks may have different nicknames or handwriting.
- Same person appearing as "Chris K." in one week and "C. K." in another.

### Stronger Future Approaches
- During review: Show top 3 fuzzy matches with confidence scores and let organizer pick with one tap.
- Levenshtein + phonetic matching (Soundex/Metaphone for names).
- "Player ID" system: Assign every player a stable internal ID in the master tracker. The website works with IDs; names are just display labels.
- Allow organizer to "merge" two names they realize are the same person, with audit.

**For tonight**: The current suggestion UI + manual override is acceptable. We can strengthen the matching algorithm next time.

---

## 3. Court Grouping, Physical Courts & Movement Rules

### Current State
- Court numbers are assigned in the review table (initially auto-grouped by count, then editable).
- We store `court` per player per week.

### Missing Logic
- **Physical court mapping**: As you mentioned, top group usually plays on physical court 3, next on 4. This is important for the "winners move up, losers move down" social dynamic.
- **Promotion / Relegation between weeks**: The site doesn't yet enforce or suggest court assignments for the *next* week based on this week's results + movement rules.
- **Irregular group sizes**: Some courts have 4 players, some 6. The point allocation (N down to 1) needs to be per actual court? Or global? (We currently do global ranking by Games Won.)

**Recommendation**: 
- Add an optional "Physical Court" field in the review table.
- Store movement history so we can later show "Player X moved up from Court 5 to Court 4".
- This data is also prerequisite for the partner/pairing analytics you mentioned earlier.

---

## 4. Score Entry & Validation Rules

### Current State
- Primary path: Photo upload → OCR → editable table → "Calculate Points using N from context".
- Secondary: Old bulk paste / manual entry (now admin-only).

### Gaps
- **Per-court vs Global ranking**: The league rule is "rank everyone who played that night globally by total games won, then assign points N, N-1, ...". We do this. But the paper sheets are grouped by court, so the organizer is mentally doing per-court ranking first. The system should make the global ranking very explicit and visible.
- **Validation during review**:
  - Sum of games won on a court should make sense.
  - Court Rank 1 should usually have the highest Games Won on that court (flag anomalies).
  - Duplicate names within the same week should be prevented or merged.
- **Incomplete sheets**: What if one court’s scores are missing or illegible? The organizer needs an easy way to mark "this court’s data is partial" and still commit the rest.
- **Dispute handling**: Easy way to attach a note or photo to a specific player’s entry for a week.

**Nice-to-have later**: A "Validate Week" button that runs a bunch of sanity checks before commit.

---

## 5. Standings Calculations & Edge Cases

### Current State
- Drop 2 lowest weeks for final total.
- Recent Form uses normalized % of that week’s max.

### Missing / Edge Cases
- **Tiebreakers**: What happens when two players have the exact same final total? Current sort is unstable. Common tiebreakers in ladders: head-to-head, total games won differential, most weeks played, etc.
- **Byes / No-shows**: How are weeks with very low attendance handled? Do we still use the actual N that night, or a season average?
- **Mid-season format changes**: The league sometimes plays sets to 7 on certain courts. The system needs to be able to record "this week used different scoring on Court X" without breaking point calculations.
- **Season reset / carry-over**: At the end of the season, do some players carry ranking into the next season? How is that modeled?

---

## 6. Workflow & Operational Robustness

### Current State
- Fully client-side (localStorage + self-contained HTML).
- Organizer can download an updated snapshot.

### Important Considerations for Real Use
- **Backup & Recovery**: If the organizer’s phone dies or they clear browser data, the only backup is the Excel master tracker + the downloaded HTML snapshots. We should make exporting a rich JSON/CSV of all raw weeks very easy.
- **Multi-device use**: What if the organizer starts the review on phone and wants to finish on laptop? Current localStorage doesn’t travel.
- **"Submit by photo" for regular players**: Currently only the organizer uses the photo tool. Regular players just WhatsApp. In the future you might want a public upload form (with rate limiting / moderation) so multiple people can submit photos and the organizer just reviews/approves.
- **Pre-week sign-up vs actual attendance**: The site currently doesn’t model the sign-up list at all. If you want to track "no-show rate" per player over the season, you’ll need that data.

---

## 7. Analytics & Future "Interesting" Features

You mentioned wanting to eventually analyze pairings:
- Who plays well with whom?
- Performance of specific doubles teams over time.

This requires storing **pairing information**, not just individual Games Won. The current per-player `gamesWon` is not enough — we need to know who was on the same side of the court for each set.

Other nice future views:
- Individual player trend lines (not just Recent Form pips).
- "Most improved" calculations.
- Court-by-court dominance stats.

---

## 8. Technical / Deployment Considerations

- Moving from single HTML file → proper hosted site (GitHub Pages, Vercel, Netlify) with proper PWA support for better phone experience.
- Using Google Sheets as the backend (read/write via API) so the organizer can keep using the tool they already love, while the website becomes a beautiful frontend.
- Adding real (but simple) authentication if the site ever becomes public or multi-organizer.

---

## Summary Recommendation (as of tonight)

**For immediate use (league night tomorrow):**
- The current state (after this commit) is good enough to be operational.
- The biggest risks are name spelling drift and manual errors in the review step — the new matching suggestions + manual override help a lot.
- Keep the Excel master tracker as the real source of truth for now.

**High-value next items when you have time/energy:**
1. Stronger name matching + "confirm canonical name" step in the review UI.
2. Ability to record Physical Court numbers and basic movement history.
3. Easy "Export full season JSON/CSV" from the admin tools (for backup and future analytics).
4. Clearer validation warnings before committing a week.

This document can live in the repo so future you (or future collaborators) know what was intentionally left out for v1.
