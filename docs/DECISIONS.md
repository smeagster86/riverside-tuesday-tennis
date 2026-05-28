[THE FULL DECISIONS.md WITH THE FOLLOWING NEW ENTRY APPENDED AT THE END BEFORE THE "Future Decision Template" SECTION:

## D-010: Review & Correct Past Weeks + Unscored Players Input for Missing Courts (May 2026)

**Date:** 2026-05 (post D-009)
**Status:** Implemented

**Context / Problem:** 
The organizer needed to correct an erroneous score in a past week and re-run the N-point allocation logic for only that week, remove a duplicate week entry, or declare that a court had played but no individual scores were recorded (so the true headcount is used for the top player's points while the unscored players get zero in stored data). There was no UI or functions for this. The photo upload drop zone was also fragile on iOS and after the cleanup.

**Decision:** 
Added a complete "Review & Correct Past Weeks" section in the admin portal (after the photo workflow). It lists committed weeks and allows:
- Loading any week's rich weekGames into an editable spreadsheet (modeled on the OCR review table, with live canonical name suggestions).
- Live "Points this week would be" preview that updates on every edit.
- A clear "Additional unscored players this week" numeric input (added only to effective N for the allocation math).
- "Save edits + re-allocate only this week" (updates only that week's slice in players.points[] and weekGames).
- Void/delete a week (with strong confirmation + backup nudge; prefer void with flag for audit).
- Global "Rebuild entire season from weekGames" nuclear option.

Photo upload was hardened (proper <label> + explicit button + robust re-init after login).

All calculations (standings, adjusted total, Recent Form, etc.) gracefully skip voided weeks.

Safety: prominent "Download full snapshot" before any edit.

**Artifacts / Implementation:**
- New HTML section in #tools (after photo workflow card).
- New JS functions: loadWeekForEditing, renderWeekEditorTable, saveWeekEditsAndReallocate (with effectiveN logic), deleteOrVoidWeek, rebuildAllFromWeekGames.
- Minor updates to existing calc/render functions for voided weeks.
- Updated Architecture notes comment.
- This D-010 entry + note in PROJECT_CONTEXT.md.

This directly solves the incomplete-sheet gap flagged in LEAGUE_LOGIC_CONSIDERATIONS.md and the user's operational need for post-entry corrections.]