# WORTDUELL — Das deutsche Kreuzwort-Duell

A browser word game (Scrabble-style crossword duel) with German dictionary validation. No build step, no server logic — a fully static site.

**Modes:** local 2-player hotseat · vs. computer (3 difficulties) · online P2P duel via 4-letter code
**Features:** 15×15 board with premium squares, classic German tile distribution (incl. Ä/Ö/Ü, 2 blanks), live word validation, best-move hints, move log, tile-bag counter, sound effects, DE/EN UI toggle, autosave/resume.

## Files

- `index.html` — the entire game (UI + logic in one file)
- `support.js` — runtime the page loads (required, same directory)
- `dict/basis.txt` — bundled German word list (~54k common forms), fetched at runtime
- `.nojekyll` — tells GitHub Pages to serve files as-is

## Rules & word validation

- First word must cover the center star (✦); every later word must connect to
  the board with no gaps. Minimum word length is 2 letters (no upper bound).
  Playing all 7 tiles in a single move earns a configurable bonus
  (`bingoBonus`, default 50).
- Every move is checked against **all** words it forms, not just the one the
  player intended: the main word in the direction it was played, *and* every
  crossing (perpendicular) word each newly placed tile creates. The move is
  only accepted if every one of those words validates.
- Nothing is invented — see "Word-list credits" below for the two source
  lists. The large auto-expanded runtime list (~685k forms) is not curated
  for Scrabble-style play: it trades curation for coverage, so it does
  contain some abbreviations and unusual short forms (e.g. "AA", "ND") that
  are technically valid dictionary entries but don't read like "real" words.
- In-game, the same information is available from the "Regeln"/"Rules"
  button on the main menu, or the "?" button next to "Menü" during a match.

## Runtime network dependencies (all optional-ish)

- **Google Fonts** (Alegreya / Alegreya Sans) — cosmetic fallback if offline
- **PeerJS** from unpkg.com + the free PeerJS cloud broker — only needed for the online P2P mode; local & CPU modes work without it
- **Full dictionary** (~685k word forms) streamed once from `cdn.jsdelivr.net/gh/hippler/german-wordlist@master/words.txt` (fallback: raw.githubusercontent.com), then cached in the browser's IndexedDB. Without it the bundled 54k list is used.

Because `index.html` fetches `dict/basis.txt`, the game must be served over HTTP(S) — opening via `file://` will run but without dictionary validation. Locally: `python3 -m http.server` in this folder.

## Publish with Claude Code

Paste this into Claude Code from inside this folder:

```
Publish this folder as a GitHub Pages site:
1. git init, commit all files (including .nojekyll).
2. Create a public GitHub repo named "wortduell" with the gh CLI and push to main.
3. Enable GitHub Pages from the main branch root:
   gh api repos/{owner}/wortduell/pages -X POST -f "source[branch]=main" -f "source[path]=/"
4. Print the published URL (https://<owner>.github.io/wortduell/) and verify it returns 200
   and that /dict/basis.txt is served.
```

No build, no framework — do not add bundlers or convert the code; deploy the files exactly as they are.

## Word-list credits

- Basis list derived from OpenSubtitles frequency lists (hermitdave/FrequencyWords, CC-BY) and Jonny-exe/German-Words-Library
- Full runtime list: hippler/german-wordlist (word list for word games, based on Markus Enzberger's list, used by Tanglet)

“Scrabble” is a trademark of its respective owners; WORTDUELL is an original implementation and design not affiliated with them.
