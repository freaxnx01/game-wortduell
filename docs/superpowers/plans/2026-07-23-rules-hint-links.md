# WORTDUELL: Rules screen, dictionary docs, multiplayer hint gating, word search links — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Rules overlay + README docs answering word-validation/dictionary-source questions, hide the Hint button outside vs-computer mode, and make played words in the move log open a Google search in a new tab.

**Architecture:** All changes live in the single `index.html` file (a custom `sc-if`/`sc-for` template mini-framework rendered by `support.js`, state held in one `Component extends DCLogic` class via `this.state` + a `renderVals()` method that returns the props consumed by the template). No new files, no build step, no dependencies.

**Tech Stack:** Vanilla JS (ES5-ish, no modules), the repo's existing `sc-if`/`sc-for`/`{{ }}` template syntax, `support.js` runtime (do not modify).

## Global Constraints

- Spec doc: `docs/superpowers/specs/2026-07-23-rules-hint-links-design.md` (approved).
- `index.html` carries a `type="text/x-dc"` bundler marker that would normally mean "generated, edit the source instead" — but this repo has no checked-in source file and no `dc-tool` CLI is available. The user explicitly approved hand-editing `index.html` directly (see spec background). Do not attempt to locate or regenerate a separate source file.
- Every new user-facing string needs both a `de` and an `en` entry in the `T` dictionary (`index.html` lines ~278-325) — this repo's localization rule applies to all UI text.
- This is a buildless stack: there is no unit-test toolchain. Each task's "test" step is a scripted Playwright smoke check (Python, `playwright.sync_api`, already confirmed installed on this machine) driving a local `python3 -m http.server` instance, per this repo's own `CLAUDE.md` ("manual in-browser playtest is the test gate").
- Match existing code style exactly: inline `style="..."` attributes, `style-hover="..."` pseudo-class attribute, the same color palette already in use (`#e0b05c`, `#cdb992`, `#4a3520`/`#3a2917` gradient card background, etc.), and the existing `fmt()`/`t()` helper patterns. Do not introduce a CSS framework, a build step, or restructure unrelated code.
- Do not modify `dict/basis.txt`, `support.js`, `version.js`, or the word lists/dictionary logic (`loadFull`, `merge`, `ensureMap`) — none of that is in scope.
- Commit messages: Conventional Commits (`feat(...)`, `docs(...)`), per this repo's `CLAUDE.md`.
- Tasks 1-3 all edit `index.html` in different regions but the same `renderVals()` return object and the same `T.de`/`T.en` dictionaries — **execute them in order (Task 1 → 2 → 3 → 4), not in parallel**, to avoid merge conflicts on those shared blocks.

---

### Task 1: Hide the Hint button outside vs-computer mode

**Files:**
- Modify: `index.html:189` (action-button grid `style` attribute)
- Modify: `index.html:192` (Hint button, wrap in `sc-if`)
- Modify: `index.html:960` (`renderVals()` — add `hintVisible`/`actionGridCols`)
- Test: `/tmp/claude-1000/wortduell-tests/test_hint_gating.py` (new scratch script, not committed)

**Interfaces:**
- Produces: `hintVisible` (boolean, `true` only when `s.mode === 'cpu'`) and `actionGridCols` (string, `'repeat(5,1fr)'` or `'repeat(4,1fr)'`) in the object returned by `renderVals()`. Later tasks do not depend on these names, but must not reintroduce a hardcoded `repeat(5,1fr)` on the same grid `div`.

- [ ] **Step 1: Change the action grid to use a dynamic column count**

In `index.html`, find (inside the `notExch` block, ~line 189):

```html
            <div style="display:grid; grid-template-columns:repeat(5,1fr); gap:7px;">
```

Replace with:

```html
            <div style="display:grid; grid-template-columns:{{ actionGridCols }}; gap:7px;">
```

- [ ] **Step 2: Wrap the Hint button in `sc-if value="{{ hintVisible }}"`**

Find (~line 192):

```html
              <button onClick="{{ onHint }}" disabled="{{ actionsDisabled }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#e0b05c; padding:9px 2px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;" style-hover="border-color:#8a6a42">{{ t.hint }}</button>
```

Replace with:

```html
              <sc-if value="{{ hintVisible }}" hint-placeholder-val="{{ true }}">
                <button onClick="{{ onHint }}" disabled="{{ actionsDisabled }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#e0b05c; padding:9px 2px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;" style-hover="border-color:#8a6a42">{{ t.hint }}</button>
              </sc-if>
```

- [ ] **Step 3: Add `hintVisible`/`actionGridCols` to `renderVals()`**

Find (~line 960):

```js
      actionsDisabled: !myTurn, exchDisabled: !myTurn || s.bag.length < 7,
```

Replace with:

```js
      actionsDisabled: !myTurn, exchDisabled: !myTurn || s.bag.length < 7,
      hintVisible: s.mode === 'cpu', actionGridCols: s.mode === 'cpu' ? 'repeat(5,1fr)' : 'repeat(4,1fr)',
```

- [ ] **Step 4: Write and run the Playwright smoke check**

```bash
mkdir -p /tmp/claude-1000/wortduell-tests
cd /home/freax/repos/github/freaxnx01/game-wortduell
(python3 -m http.server 8711 >/tmp/claude-1000/wortduell-tests/server.log 2>&1 &)
sleep 1
```

Write `/tmp/claude-1000/wortduell-tests/test_hint_gating.py`:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(args=["--no-sandbox"])
    page = browser.new_page()
    page.goto("http://localhost:8711/index.html")
    page.wait_for_timeout(1500)

    # Local hotseat mode: Hint must be absent
    page.click("text=Zu zweit am Gerät")
    page.wait_for_timeout(500)
    assert page.locator("text=Tipp").count() == 0, "Hint button should be hidden in local mode"
    print("PASS: hint hidden in local mode")

    page.goto("http://localhost:8711/index.html")
    page.wait_for_timeout(1500)

    # CPU mode: Hint must be present
    page.click("text=Spiel starten")
    page.wait_for_timeout(500)
    assert page.locator("text=Tipp").count() == 1, "Hint button should be visible in CPU mode"
    print("PASS: hint visible in cpu mode")

    browser.close()
```

Run: `python3 /tmp/claude-1000/wortduell-tests/test_hint_gating.py`
Expected output:
```
PASS: hint hidden in local mode
PASS: hint visible in cpu mode
```

- [ ] **Step 5: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/game-wortduell
git add index.html
git commit -m "feat(gameplay): hide Hint button outside vs-computer mode

The Hint button previously worked on any turn in every mode, including
local hotseat and P2P versus another human — giving an unfair
in-match advantage the feature was never meant to grant there."
```

---

### Task 2: Clickable Google-search links for played words in the move log

**Files:**
- Modify: `index.html:571` (`applyMove` — log entry for a played move)
- Modify: `index.html:594` (`doPass` — log entry for a pass)
- Modify: `index.html:618` (`doExchange` — log entry for an exchange)
- Modify: `index.html:786` (`hintMove` — log entry for a hint use)
- Modify: `index.html:965` (`renderVals()` — `logRows` mapping)
- Modify: `index.html:202-208` (move-log template block)
- Test: `/tmp/claude-1000/wortduell-tests/test_log_links.py` (new scratch script, not committed)

**Interfaces:**
- Consumes: nothing from Task 1.
- Produces: each entry in `this.state.log` now has shape `{ k, who, words: string[], real: boolean, pts }` instead of `{ k, who, txt: string, pts }`. `real: true` only for entries created in `applyMove` (an actual played word); `real: false` for pass/exchange/hint-used entries. `renderVals()`'s `logRows` maps this into `{ who, words: [{str, prefix, href}], pts }` for the template, where `href` is `null` unless the parent entry has `real: true`.

- [ ] **Step 1: Change the played-move log entry to store a `words` array**

Find (~line 571, inside `applyMove`):

```js
    const log = [{ k: this.uid++, who: s.names[player], txt: r.words.map(w => w.str).join(' · '), pts: '+' + gained }, ...s.log];
```

Replace with:

```js
    const log = [{ k: this.uid++, who: s.names[player], words: r.words.map(w => w.str), real: true, pts: '+' + gained }, ...s.log];
```

- [ ] **Step 2: Change the pass, exchange, and hint log entries to the same shape**

Find (~line 594, inside `doPass`):

```js
    const log = [{ k: this.uid++, who: s.names[player], txt: t.passedTxt, pts: '' }, ...s.log];
```

Replace with:

```js
    const log = [{ k: this.uid++, who: s.names[player], words: [t.passedTxt], real: false, pts: '' }, ...s.log];
```

Find (~line 618, inside `doExchange`):

```js
    const log = [{ k: this.uid++, who: s.names[player], txt: this.fmt(t.exchangedTxt, { n: removed.length }), pts: '' }, ...s.log];
```

Replace with:

```js
    const log = [{ k: this.uid++, who: s.names[player], words: [this.fmt(t.exchangedTxt, { n: removed.length })], real: false, pts: '' }, ...s.log];
```

Find (~line 786, inside `hintMove`):

```js
      const log = [{ k: this.uid++, who: this.state.names[rackIdx], txt: t.hintTxt, pts: '' }, ...this.state.log];
```

Replace with:

```js
      const log = [{ k: this.uid++, who: this.state.names[rackIdx], words: [t.hintTxt], real: false, pts: '' }, ...this.state.log];
```

- [ ] **Step 3: Update `logRows` in `renderVals()` to build per-word link objects**

Find (~line 965):

```js
      logRows: s.log.slice(0, 40).map(l => ({ who: l.who, txt: l.txt, pts: l.pts })),
```

Replace with:

```js
      logRows: s.log.slice(0, 40).map(l => ({
        who: l.who,
        words: l.words.map((str, i) => ({
          str,
          prefix: i === 0 ? '' : ' · ',
          href: l.real ? 'https://www.google.com/search?q=' + encodeURIComponent(str) : null
        })),
        pts: l.pts
      })),
```

- [ ] **Step 4: Update the move-log template to render each word as a link (or plain text)**

Find (~lines 202-208):

```html
            <sc-for list="{{ logRows }}" as="lr" hint-placeholder-count="2">
              <div style="display:flex; gap:8px; font-size:13.5px; align-items:baseline;">
                <span style="color:#a8946f; white-space:nowrap;">{{ lr.who }}</span>
                <span style="color:#e6dcc4; flex:1; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;">{{ lr.txt }}</span>
                <span style="color:#d9a84e; font-weight:700; white-space:nowrap;">{{ lr.pts }}</span>
              </div>
            </sc-for>
```

Replace with:

```html
            <sc-for list="{{ logRows }}" as="lr" hint-placeholder-count="2">
              <div style="display:flex; gap:8px; font-size:13.5px; align-items:baseline;">
                <span style="color:#a8946f; white-space:nowrap;">{{ lr.who }}</span>
                <span style="color:#e6dcc4; flex:1; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;">
                  <sc-for list="{{ lr.words }}" as="w" hint-placeholder-count="8">
                    {{ w.prefix }}<sc-if value="{{ w.href }}" hint-placeholder-val="{{ false }}"><a href="{{ w.href }}" target="_blank" rel="noopener" style="color:#e6dcc4; text-decoration:underline dotted;" style-hover="color:#f0c97e">{{ w.str }}</a></sc-if><sc-if value="{{ !w.href }}" hint-placeholder-val="{{ true }}"><span>{{ w.str }}</span></sc-if>
                  </sc-for>
                </span>
                <span style="color:#d9a84e; font-weight:700; white-space:nowrap;">{{ lr.pts }}</span>
              </div>
            </sc-for>
```

- [ ] **Step 5: Write and run the Playwright smoke check**

The dictionary needs network access to validate a real played word, so this check exercises the plain-text (non-link) path via a pass action — the part fully reachable without a live dictionary — and confirms no console errors from the data-model change. Manually verify the link path once (Step 6).

Write `/tmp/claude-1000/wortduell-tests/test_log_links.py`:

```python
from playwright.sync_api import sync_playwright

errors = []
with sync_playwright() as p:
    browser = p.chromium.launch(args=["--no-sandbox"])
    page = browser.new_page()
    page.on("pageerror", lambda exc: errors.append(str(exc)))
    page.goto("http://localhost:8711/index.html")
    page.wait_for_timeout(1500)
    page.click("text=Zu zweit am Gerät")
    page.wait_for_timeout(500)
    page.click("text=Passen")   # arms the confirm
    page.wait_for_timeout(300)
    page.click("text=Sicher?")  # confirms the pass -> writes a non-real log entry
    page.wait_for_timeout(500)

    row_text = page.locator("text=passt").first.inner_text()
    assert "passt" in row_text, "expected the pass message to render as plain text"
    assert page.locator("a:has-text('passt')").count() == 0, "pass message must not be a link"
    print("PASS: non-word log entry renders as plain text, no link")

    assert not errors, f"page errors: {errors}"
    print("PASS: no console/page errors")
    browser.close()
```

Run: `python3 /tmp/claude-1000/wortduell-tests/test_log_links.py`
Expected output:
```
PASS: non-word log entry renders as plain text, no link
PASS: no console/page errors
```

- [ ] **Step 6: Manual check of the real link path**

In a browser (not headless), open `http://localhost:8711/index.html`, start "Gegen den Computer", play any legal opening word, and confirm in the move log that the word you played is underlined and clicking it opens a Google search for that word in a new tab.

- [ ] **Step 7: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/game-wortduell
git add index.html
git commit -m "feat(ui): make played words in the move log open a Google search

Each word a move actually formed is now a link (target=_blank) to a
Google search for that word; pass/exchange/hint status messages stay
plain text since they aren't played words."
```

---

### Task 3: Rules overlay (menu button + in-game button + modal + translations)

**Files:**
- Modify: `index.html:265-274` (state — add `showRules`)
- Modify: `index.html:278-324` (`T.de`/`T.en` — add rules copy)
- Modify: `index.html:44-49` (menu screen — add "Regeln"/"Rules" button)
- Modify: `index.html:146-148` (in-game header — add "?" button)
- Modify: `index.html` (new Rules modal block, inserted after the `showOver` modal, before the closing root `</div>`)
- Modify: `index.html:928-979` (`renderVals()` — add `showRules`, `onOpenRules`, `onCloseRules`, `onStopProp`, `rulesHowToBody`)
- Test: `/tmp/claude-1000/wortduell-tests/test_rules_overlay.py` (new scratch script, not committed)

**Interfaces:**
- Consumes: nothing from Tasks 1-2 directly, but edits the same `renderVals()` return object and `T.de`/`T.en` dictionaries those tasks already touched — run this task after Tasks 1-2 are committed.
- Produces: `showRules` (boolean state field), `onOpenRules`/`onCloseRules`/`onStopProp` (functions in `renderVals()`), `rulesHowToBody` (pre-formatted string in `renderVals()`). No later task depends on these.

- [ ] **Step 1: Add `showRules` to initial state**

Find (~line 271):

```js
    handover: false, blankPick: null, hint: null, thinking: false, passArm: false, shake: false,
```

Replace with:

```js
    handover: false, blankPick: null, hint: null, thinking: false, passArm: false, shake: false, showRules: false,
```

- [ ] **Step 2: Add German rules copy to `T.de`**

Find (~line 300, the last line of the `de` block):

```js
      sound: 'Ton an', soundOff: 'Ton aus', lblDL: '2·B', lblTL: '3·B', lblDW: '2·W', lblTW: '3·W'
```

Replace with:

```js
      sound: 'Ton an', soundOff: 'Ton aus', lblDL: '2·B', lblTL: '3·B', lblDW: '2·W', lblTW: '3·W',
      rulesBtn: 'Regeln', rulesTitle: 'Regeln', close: 'Schließen',
      rulesHowToT: 'So wird gespielt',
      rulesHowToBody: 'Das erste Wort muss das Sternfeld (✦) in der Mitte bedecken. Jedes weitere Wort muss an ein bereits liegendes Wort anschließen — Lücken sind nicht erlaubt. Ein Wort braucht mindestens zwei Buchstaben. Wer alle 7 Steine auf einmal legt, bekommt einen Bonus von {n} Punkten. 2·B/3·B verdoppeln bzw. verdreifachen den Buchstabenwert, 2·W/3·W den Wortwert — jedes Feld zählt nur beim ersten Belegen.',
      rulesValidT: 'Wortprüfung',
      rulesValidBody: 'Bei jedem Zug werden alle neu entstehenden Wörter geprüft — das Hauptwort in Legerichtung UND jedes einzelne Wort, das quer dazu durch die neuen Steine entsteht. Nur wenn wirklich jedes dieser Wörter im Wörterbuch steht, wird der Zug akzeptiert. Kurze Wörter mit zwei Buchstaben sind ausdrücklich erlaubt.',
      rulesDictT: 'Wörterbuch-Quelle',
      rulesDictBody: 'Kein Wort wird erfunden. Die Basisliste (~54.000 Formen) stammt aus Frequenzlisten von OpenSubtitles sowie der German-Words-Library (Jonny-exe). Beim Start wird zusätzlich automatisch eine große Liste (~685.000 Formen, hippler/german-wordlist, bekannt aus dem Wortspiel Tanglet) nachgeladen. Diese große Liste ist nicht für Scrabble-artige Spiele kuratiert — sie deckt viel ab, aber sie enthält auch Abkürzungen und ungewöhnliche Kurzformen (z. B. „AA“, „ND“), die technisch gültig sind, aber selten wie „echte“ Wörter wirken.'
```

- [ ] **Step 3: Add English rules copy to `T.en`**

Find (~line 323, the last line of the `en` block):

```js
      sound: 'Sound on', soundOff: 'Muted', lblDL: 'DL', lblTL: 'TL', lblDW: 'DW', lblTW: 'TW'
```

Replace with:

```js
      sound: 'Sound on', soundOff: 'Muted', lblDL: 'DL', lblTL: 'TL', lblDW: 'DW', lblTW: 'TW',
      rulesBtn: 'Rules', rulesTitle: 'Rules', close: 'Close',
      rulesHowToT: 'How to play',
      rulesHowToBody: 'The first word must cover the star square (✦) in the center. Every following word must connect to a word already on the board — no gaps allowed. A word needs at least two letters. Playing all 7 tiles in one move earns a {n}-point bonus. 2·B/3·B double or triple a letter’s value, 2·W/3·W double or triple the whole word’s value — each square only counts the first time it’s covered.',
      rulesValidT: 'Word validation',
      rulesValidBody: 'Every move checks all words it newly forms — the main word in the direction it was played AND every single word formed crosswise through the new tiles. The move is only accepted if every one of those words is in the dictionary. Short, two-letter words are explicitly allowed.',
      rulesDictT: 'Dictionary source',
      rulesDictBody: 'Nothing is invented. The base list (~54k forms) comes from OpenSubtitles frequency lists and the German-Words-Library (Jonny-exe). On startup, a large list (~685k forms, hippler/german-wordlist, used by the word game Tanglet) is streamed in automatically. That large list isn’t curated for Scrabble-style play — it covers a lot, but also includes abbreviations and unusual short forms (e.g. “AA”, “ND”) that are technically valid but rarely feel like “real” words.'
```

- [ ] **Step 4: Add the "Regeln"/"Rules" button to the main menu**

Find (~line 44-49):

```html
      <div style="display:flex; gap:10px; flex-wrap:wrap; justify-content:center; margin-top:22px; align-items:center;">
        <input value="{{ nameP1 }}" onChange="{{ onNameP1 }}" placeholder="{{ t.ph1 }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#f0e6d2; padding:9px 12px; font:inherit; font-size:15px; width:150px;">
        <input value="{{ nameP2 }}" onChange="{{ onNameP2 }}" placeholder="{{ t.ph2 }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#f0e6d2; padding:9px 12px; font:inherit; font-size:15px; width:150px;">
        <button onClick="{{ onToggleLang }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#e0b05c; padding:9px 14px; font:inherit; font-size:14px; font-weight:700; cursor:pointer;">{{ langLabel }}</button>
        <button onClick="{{ onToggleSound }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#e0b05c; padding:9px 14px; font:inherit; font-size:14px; font-weight:700; cursor:pointer;">{{ soundLabel }}</button>
      </div>
```

Replace with:

```html
      <div style="display:flex; gap:10px; flex-wrap:wrap; justify-content:center; margin-top:22px; align-items:center;">
        <input value="{{ nameP1 }}" onChange="{{ onNameP1 }}" placeholder="{{ t.ph1 }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#f0e6d2; padding:9px 12px; font:inherit; font-size:15px; width:150px;">
        <input value="{{ nameP2 }}" onChange="{{ onNameP2 }}" placeholder="{{ t.ph2 }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#f0e6d2; padding:9px 12px; font:inherit; font-size:15px; width:150px;">
        <button onClick="{{ onToggleLang }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#e0b05c; padding:9px 14px; font:inherit; font-size:14px; font-weight:700; cursor:pointer;">{{ langLabel }}</button>
        <button onClick="{{ onToggleSound }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#e0b05c; padding:9px 14px; font:inherit; font-size:14px; font-weight:700; cursor:pointer;">{{ soundLabel }}</button>
        <button onClick="{{ onOpenRules }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#e0b05c; padding:9px 14px; font:inherit; font-size:14px; font-weight:700; cursor:pointer;">{{ t.rulesBtn }}</button>
      </div>
```

- [ ] **Step 5: Add the "?" button to the in-game header**

Find (~line 146-148):

```html
          <button onClick="{{ onToggleLang }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:7px; color:#e0b05c; padding:6px 10px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;">{{ langLabel }}</button>
          <button onClick="{{ onToggleSound }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:7px; color:#e0b05c; padding:6px 10px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;">{{ soundLabel }}</button>
          <button onClick="{{ onLeaveGame }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:7px; color:#cdb992; padding:6px 10px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;" style-hover="color:#f0e6d2">{{ t.menuBtn }}</button>
```

Replace with:

```html
          <button onClick="{{ onToggleLang }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:7px; color:#e0b05c; padding:6px 10px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;">{{ langLabel }}</button>
          <button onClick="{{ onToggleSound }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:7px; color:#e0b05c; padding:6px 10px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;">{{ soundLabel }}</button>
          <button onClick="{{ onOpenRules }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:7px; color:#e0b05c; padding:6px 10px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;" style-hover="border-color:#8a6a42">?</button>
          <button onClick="{{ onLeaveGame }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:7px; color:#cdb992; padding:6px 10px; font:inherit; font-size:12.5px; font-weight:700; cursor:pointer;" style-hover="color:#f0e6d2">{{ t.menuBtn }}</button>
```

- [ ] **Step 6: Add the Rules modal, after the `showOver` modal block**

Find (~lines 258-260, the end of the `showOver` `sc-if` block):

```html
      </div>
    </div>
  </sc-if>
</div>
</x-dc>
```

Replace with:

```html
      </div>
    </div>
  </sc-if>

  <sc-if value="{{ showRules }}" hint-placeholder-val="{{ false }}">
    <div style="position:fixed; inset:0; z-index:80; background:rgba(28,18,10,.74); backdrop-filter:blur(14px); display:flex; align-items:center; justify-content:center; padding:20px; box-sizing:border-box;" onClick="{{ onCloseRules }}">
      <div style="background:linear-gradient(180deg,#4a3520,#3a2917); border-radius:18px; padding:26px 28px; max-width:560px; width:92vw; max-height:82vh; overflow-y:auto; box-sizing:border-box; box-shadow:0 24px 60px rgba(0,0,0,.6);" onClick="{{ onStopProp }}">
        <div style="display:flex; align-items:center; justify-content:space-between; margin-bottom:14px;">
          <div style="font-family:Alegreya,serif; font-weight:800; font-size:26px; color:#f3ead6;">{{ t.rulesTitle }}</div>
          <button onClick="{{ onCloseRules }}" style="background:rgba(0,0,0,.28); border:1px solid #6b4f33; border-radius:8px; color:#cdb992; padding:7px 14px; font:inherit; font-size:13px; cursor:pointer;" style-hover="color:#f0e6d2">{{ t.close }}</button>
        </div>
        <div style="display:flex; flex-direction:column; gap:16px;">
          <div>
            <div style="font-weight:700; font-size:15.5px; color:#e0b05c; margin-bottom:5px;">{{ t.rulesHowToT }}</div>
            <div style="font-size:14px; line-height:1.55; color:#e6dcc4;">{{ rulesHowToBody }}</div>
          </div>
          <div>
            <div style="font-weight:700; font-size:15.5px; color:#e0b05c; margin-bottom:5px;">{{ t.rulesValidT }}</div>
            <div style="font-size:14px; line-height:1.55; color:#e6dcc4;">{{ t.rulesValidBody }}</div>
          </div>
          <div>
            <div style="font-weight:700; font-size:15.5px; color:#e0b05c; margin-bottom:5px;">{{ t.rulesDictT }}</div>
            <div style="font-size:14px; line-height:1.55; color:#e6dcc4;">{{ t.rulesDictBody }}</div>
          </div>
        </div>
      </div>
    </div>
  </sc-if>
</div>
</x-dc>
```

- [ ] **Step 7: Add `showRules`, the open/close handlers, and `rulesHowToBody` to `renderVals()`**

Find (~line 977-978, the end of the returned object):

```js
      onLeaveGame: () => this.leaveGame(),
      onToMenu: () => this.leaveGame()
    };
```

Replace with:

```js
      onLeaveGame: () => this.leaveGame(),
      onToMenu: () => this.leaveGame(),
      showRules: s.showRules,
      onOpenRules: () => this.setState({ showRules: true }),
      onCloseRules: () => this.setState({ showRules: false }),
      onStopProp: e => e.stopPropagation(),
      rulesHowToBody: this.fmt(t.rulesHowToBody, { n: bb })
    };
```

(`bb` is already defined earlier in `renderVals()` as `const bb = this.props.bingoBonus ?? 50;` — no new variable needed.)

- [ ] **Step 8: Write and run the Playwright smoke check**

Write `/tmp/claude-1000/wortduell-tests/test_rules_overlay.py`:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(args=["--no-sandbox"])
    page = browser.new_page()

    # Menu entry point
    page.goto("http://localhost:8711/index.html")
    page.wait_for_timeout(1500)
    page.click("text=Regeln")
    page.wait_for_timeout(300)
    assert page.locator("text=Wortprüfung").count() == 1, "Rules modal should show the word-validation section"
    page.click("text=Schließen")
    page.wait_for_timeout(300)
    assert page.locator("text=Wortprüfung").count() == 0, "Rules modal should close"
    print("PASS: rules overlay opens/closes from the main menu")

    # In-game entry point, mid-match, without leaving the board
    page.click("text=Zu zweit am Gerät")
    page.wait_for_timeout(500)
    page.click("button:has-text('?')")
    page.wait_for_timeout(300)
    assert page.locator("text=Wortprüfung").count() == 1, "Rules modal should open from the in-game '?' button"
    assert page.locator("[data-screen-label='Spielbrett']").count() == 1, "board must still be mounted underneath (no leave-to-menu)"
    print("PASS: rules overlay opens in-game without leaving the match")

    browser.close()
```

Run: `python3 /tmp/claude-1000/wortduell-tests/test_rules_overlay.py`
Expected output:
```
PASS: rules overlay opens/closes from the main menu
PASS: rules overlay opens in-game without leaving the match
```

- [ ] **Step 9: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/game-wortduell
git add index.html
git commit -m "feat(ui): add in-game Rules overlay (DE/EN)

Explains how to play, that every crossing word (not just the main
word) is validated and 2-letter words are allowed, and names the
dictionary sources plus a caveat that the large auto-expanded list
isn't curated for Scrabble-style play. Reachable from the main menu
and, without leaving an in-progress match, via a new '?' button next
to Menü."
```

---

### Task 4: README.md — document rules, word validation, and dictionary source

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: nothing from Tasks 1-3 (pure documentation; can technically run any time, but sequenced last here since it references the same facts the Rules overlay in Task 3 states, to keep the two in sync while writing).

- [ ] **Step 1: Add a "Rules & word validation" section**

Find, in `README.md`:

```markdown
## Runtime network dependencies (all optional-ish)
```

Replace with:

```markdown
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
```

- [ ] **Step 2: Read the file back to confirm the section landed correctly**

```bash
grep -n "Rules & word validation" -A 20 /home/freax/repos/github/freaxnx01/game-wortduell/README.md
```

Expected: the new section prints, followed immediately by the existing "Runtime network dependencies" heading.

- [ ] **Step 3: Commit**

```bash
cd /home/freax/repos/github/freaxnx01/game-wortduell
git add README.md
git commit -m "docs: document rules, word validation, and dictionary source"
```

---

## Self-Review Notes

- **Spec coverage:** Rules overlay (§1) → Task 3; README (§2) → Task 4; Hint gating (§3) → Task 1; move-log links (§4) → Task 2. All four spec sections have a task.
- **Placeholder scan:** none — every step has literal find/replace code.
- **Type consistency:** `words`/`real` log-entry shape is introduced once (Task 2, Step 1-2) and consumed once (Task 2, Step 3) with matching field names; `hintVisible`/`actionGridCols` (Task 1) and `showRules`/`onOpenRules`/`onCloseRules`/`onStopProp`/`rulesHowToBody` (Task 3) are each defined and used within the same task, no cross-task name drift.
