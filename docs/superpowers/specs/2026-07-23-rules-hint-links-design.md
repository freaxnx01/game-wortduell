# WORTDUELL: Rules screen, dictionary docs, multiplayer hint gating, word search links

Date: 2026-07-23

## Background

Player feedback from screenshots of a live match raised four questions and two
feature requests:

1. Does word validation check both horizontal and vertical (crossing) words? —
   Yes, already correct (`evalPlacements()` in `index.html`, ~line 450: it
   collects the main word plus every perpendicular crossing word formed by
   the new placement and requires `allValid` before accepting the move). No
   code change needed, but this isn't documented anywhere a player can see.
2. Are 2-letter words allowed? — Yes, by design (`errMin2` only enforces a
   *minimum* of 2 letters). Not documented for players.
3. What's the word source? Is anything invented? — Never invented. Documented
   in `README.md` already (OpenSubtitles frequency list +
   Jonny-exe/German-Words-Library for the ~54k bundled base list,
   `hippler/german-wordlist` for the ~685k runtime expansion), but not visible
   from inside the game, and the README doesn't mention that the large
   auto-expanded list can include low-quality entries (abbreviations, loan
   forms) since it isn't curated for Scrabble-style play.
4. Feature: a Rules screen in-game plus expanded README covering the above.
5. Feature: disable the Hint button in multiplayer — it currently renders and
   works whenever it's the current player's turn in *every* mode, including
   local hotseat and P2P versus another human, which is an unfair advantage
   the CPU-mode hint feature was never meant to grant.
6. Feature: make played words in the move log clickable, opening a Google
   search for that word in a new tab.

## Scope

Repo: `game-wortduell` (single-file `index.html`, template mini-framework via
`support.js`, state in one top-level component class). Changes:

1. New "Rules" modal overlay.
2. README additions.
3. Hint button hidden (not just disabled) outside `cpu` mode.
4. Move-log word entries become clickable Google-search links.

No backend, no build step, no new dependencies — everything is static
HTML/JS edits consistent with the existing file.

## 1. Rules overlay

**Entry points:**
- A new "Regeln" / "Rules" button on the main menu screen, placed near the
  existing `rulesHint` line (`isMenu` block, ~line 79).
- A new small "?" icon button in-game, next to the existing "Menü" button
  (~line 148). Unlike "Menü" (which calls `onLeaveGame` and exits to the main
  menu), the "?" button only toggles `showRules: true` — it does not touch
  game state, so it's safe to open mid-match without any leave/resume
  round-trip and works identically for P2P (no need to coordinate with the
  opponent).

**Rendering:** a new `sc-if value="{{ showRules }}"` block using the same
fixed-overlay pattern as the existing `showHandover` / `showBlank` / `showOver`
modals (`position:fixed; inset:0`, dark blurred backdrop, centered card).
Closable via an explicit close button and by clicking the backdrop (matching
`showBlank`'s `onClick="{{ onBlankCancel }}"` pattern). Scrollable content
(`max-height` + `overflow-y:auto`) since the copy is longer than other modals.

**Content**, three sections, both DE and EN entries in the existing `t()`
translation dictionaries:

- **How to play** — first word must cover ★, every subsequent word must
  connect to the board with no gaps, minimum 2 letters per word, bonus for
  playing all 7 tiles at once, and the board premium legend (2·B/3·B =
  letter premium, 2·W/3·W = word premium). This expands on the existing
  `rulesHint` copy rather than replacing it.
- **Word validation** — states explicitly: every word a placement forms —
  the main word *and* every crossing (perpendicular) word — must be a valid
  dictionary word, or the move is rejected. Short (2-letter) words are
  allowed.
- **Dictionary source** — names both sources (OpenSubtitles frequency list +
  Jonny-exe/German-Words-Library for the base list; hippler/german-wordlist,
  the list used by Tanglet, for the runtime expansion) and a short caveat:
  the large auto-expanded list trades curation for coverage, so some
  obscure or abbreviation-like entries may validate as playable words.

New translation keys needed in both `T.de` and `T.en` (naming illustrative,
final keys chosen during implementation): `rulesBtn`, `rulesTitle`,
`rulesHowToT`, `rulesHowToBody`, `rulesValidT`, `rulesValidBody`,
`rulesDictT`, `rulesDictBody`, `close`.

## 2. README.md

Add a "Rules & word validation" section mirroring the Rules overlay content
(how to play, crossing-word validation + 2-letter words allowed, dictionary
sources + the curation-vs-coverage caveat). Keep the existing "Word-list
credits" section as-is; the new section links to it rather than duplicating
credits.

## 3. Hint button hidden in multiplayer

Currently (~line 192) the Hint button always renders inside the
`notExch` action grid (`grid-template-columns:repeat(5,1fr)`: Pass, Exchange,
Hint, Shuffle, Recall) and is only disabled via `actionsDisabled` (i.e.
disabled when it isn't your turn — mode-independent).

Change: wrap the Hint button in `sc-if value="{{ hintVisible }}"` where
`hintVisible` is computed as `s.mode === 'cpu'`. When hidden, the grid drops
to 4 columns — compute `actionGridCols` (`'repeat(5,1fr)'` vs
`'repeat(4,1fr)'`) alongside `hintVisible` and bind it to the grid container's
`grid-template-columns` inline style instead of the hardcoded `repeat(5,1fr)`.

No change to `hintMove()` logic itself — it's simply unreachable in
local/P2P modes once the button is gone.

## 4. Clickable words in the move log

**Current data model** (~line 571): a log entry is
`{ k, who, txt: r.words.map(w => w.str).join(' · '), pts }` — `txt` is a
single pre-joined string, so individual words aren't addressable in the
template.

**New data model:** each log entry carries a `words` array of
`{ str, href }` instead of a flat `txt` string:
- For a played-word entry: one item per word in `r.words`, each
  `href: 'https://www.google.com/search?q=' + encodeURIComponent(w.str)`.
- For pass/exchange/hint-used status messages (`passedTxt`, `exchangedTxt`,
  `hintTxt`): a single-item array `[{ str: <message>, href: null }]` — no
  search link, since it's not a played word.

Each word item also carries a precomputed `prefix` (`''` for the first word
in the row, `' · '` for subsequent ones) so the template can render the
separator as plain interpolated text without needing index-comparison
expressions.

**Template** (~line 202): `logRows` gains a nested `sc-for` over `lr.words`
(the engine's `walkFor` already merges parent scope into nested `sub` scope,
so nesting is safe — confirmed via `support.js`). Per word:
`{{ w.prefix }}` followed by `<sc-if value="{{ w.href }}">` → an
`<a href="{{ w.href }}" target="_blank" rel="noopener">{{ w.str }}</a>`, with
an `<sc-if value="{{ !w.href }}">` fallback rendering a plain
`<span>{{ w.str }}</span>` for non-word messages.

(Hint-disabling in multiplayer removes the `hintTxt` log-entry case for
local/P2P at the source, but the plain-span path stays needed for
`passedTxt`/`exchangedTxt`, and hints remain loggable in `cpu` mode.)

## Out of scope

- Full scoring/letter-value reference table in the Rules screen (kept
  implicit in the existing rack/board UI, per earlier discussion).
- Clickable words in the end-of-game summary (`overRows`) — move-log only.
- Any change to the actual word list / dictionary contents.
