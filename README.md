---
name: Riverside Tuesday Night Doubles
slug: riverside-tuesday-tennis
---

# Riverside Tuesday Night Doubles League

**Live site:** [https://riverside-tuesday-tennis.vercel.app/](https://riverside-tuesday-tennis.vercel.app/)

Competitive doubles tennis ladder at Riverside Badminton & Tennis Club (Saskatoon).  
Season: May – September 2026 (22 weeks).

## Quick Start (for players)

Just open the live link above on your phone or computer. No login needed for standings.

**Admin / Organizer (Adrian):**
- Click the **"Admin"** pill in the top-right corner.
- Username: `tennisadmin`
- Password: `riverside306!`
- This unlocks the photo upload + review workflow (client-side only — nothing leaves your device).

## What This Repo Contains (Why Everything Is Here)

This repository is intentionally **self-contained and recoverable**. If your computer dies, a builder window gets erased, or you open Grok on a completely different machine, you can:

1. `git clone https://github.com/smeagster86/riverside-tuesday-tennis.git`
2. Open `index.html` in any browser
3. Have **full context**: the app + every planning decision + the original score sheet photos + the master player spreadsheet + ground-truth parsed data.

### Repository Structure

```
riverside-tuesday-tennis/
├── index.html                 # The complete single-file app (Tailwind + Tesseract OCR)
├── README.md                  # You are here
├── LEAGUE_LOGIC_CONSIDERATIONS.md   # Zoomed-out thinking on league operations, edge cases, future needs
├── .gitignore
│
├── data/                      # Seed / historical data (JSON)
├── players-initial.json
│
├── score-sheets/              # Raw source material (the heart of recoverability)
│   └── week-3/
│       ├── page1.jpeg ... page3.jpeg     # Actual handwritten score sheets from the night
│       ├── Actual Score.png              # The clean target table format
│       ├── RiversideMasterTracker.xlsx   # Master player list + historical scores (source of truth for names)
│       ├── week3_parsed_review.xlsx      # Detailed parsing notes, ground truth, court cues, analysis tabs
│       ├── week3_ground_truth.*          # Structured transcription used to tune OCR/parser
│       └── *.py                            # Helper scripts used during analysis
│
└── (vercel.json, middleware.js, package.json — deployment config)
```

## The Admin Photo → Standings Workflow

1. Organizer receives 3–4 WhatsApp photos of the paper score sheets.
2. Opens the site on phone → logs in as Admin.
3. Fills two quick context questions (# courts, # players who actually played → this sets **N** for point allocation).
4. Drops the photos into the big target area.
5. Tesseract.js runs 100% in the browser → produces an editable table (Court | Player | Sets | Games Won | Court Rank).
6. Organizer corrects OCR mistakes and sees live **canonical name suggestions** pulled from `RiversideMasterTracker.xlsx` data.
7. One big green button: "LOOKS GOOD — CALCULATE POINTS & UPDATE STANDINGS".
8. Points are calculated using real league rules (global ranking by games won, N down to 1). Standings update instantly. Data saved to localStorage.

All of the above logic, name-matching rules, normalized "Recent Form" (the pips that handle wildly varying weekly attendance), and the rich data model (court groupings + per-set scores for future partner analytics) live in `index.html`.

## Why This Structure Exists

Early in the project we realized that a tennis ladder run from paper score sheets has many subtle operational details:
- Variable attendance (N) makes raw "trending" numbers misleading → we normalize per-week.
- Player name drift ("Chris K." vs "Christopher" vs "C K") across weeks.
- Only players who actually recorded a score should appear in public standings (sign-ups who never showed are noise).
- Court groupings and physical court numbers matter for movement rules and future pairing stats.
- The organizer needs an extremely phone-friendly flow because scores arrive as photos on WhatsApp.

The documents and raw files in this repo capture all of that thinking so nothing is lost when we pick the project back up on a different device or after a long break.

## Recovering / Continuing Work From Another Computer

1. Clone or download this repo.
2. Open `index.html` — the current season data + Week 3 is pre-loaded.
3. All reference photos and the master Excel are present for re-running OCR experiments or manual transcription.
4. The `LEAGUE_LOGIC_CONSIDERATIONS.md` file contains the broader design thoughts we discussed.
5. When you make improvements, just commit + push. Vercel will redeploy the site automatically.

## Tech Notes

- Single-file client-side app (no backend).
- Tailwind via CDN, Tesseract.js for in-browser OCR (privacy).
- Data persists in browser localStorage on the device that does the admin work.
- For permanent backup/export there is a "Download full site" button inside the app.

## License / Club

Built for the Riverside Badminton & Tennis Club Tuesday Night Doubles league.

Questions? Message the organizer via the WhatsApp link in the app.

---

**Last major update:** Week 3 data + full intuitive admin photo portal + all supporting artifacts centralized for recoverability.