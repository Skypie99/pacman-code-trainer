# Ghost Code — Agent & Developer Guide

A vanilla HTML/JS **terminal-command trainer** for memorizing **Claude Code**, **Mac terminal**, and **Git commands**. The "terminal, not arcade" redesign shipped — keep the calm, modern identity. Built so Sky learns by doing — keep diffs small and understandable, don't over-engineer, don't add unasked features.

## The Game

Guide the Phantom — a spectral terminal cursor — and capture the right command token. Capture the correct one and it's chomped; aim at a wrong one and a spirit is lost. Two modes: **Arcade** (survive on 3 spirits) and **Learn** (drill cards with explanations). Single-player, browser-based, zero dependencies.

**Play it:** Live at https://ghostcode.skypistudio.com. To run locally, serve the folder (e.g. `cd ~/Games/pacman-code-trainer && python3 -m http.server 8000`) and open `http://localhost:8000`.

---

## Stack

**Zero build, no framework, no TypeScript, no database.**

- **Single file:** `index.html` (~3.3k lines, inline CSS+JS — the `<script>` boundaries drift as the file grows; always re-derive them with the grep in the Green Gate below, never hardcode a line range)
- **Card deck:** `cards.js` (56 cards, `window.CARDS` array, 284 lines)
- **Test:** `test/cards.test.js` (zero-dependency validator, catches card integrity bugs)
- **CI:** `.github/workflows/ci.yml` (mirrors the local green gate)
- **Data layer:** `localStorage['gc.v1']` only (no network, no DB, no auth, no PII)

---

## File Map

| File | Role |
|---|---|
| `index.html` | Whole game: HTML structure, inline CSS (calm dark terminal palette), inline JS game loop |
| `cards.js` | Flashcard deck: 56 card objects. Add a card by copying a `{}` block, change the fields, save. Refresh the tab. |
| `test/cards.test.js` | Card validator: checks for missing fields, duplicate IDs, bad decoys, empty strings, unknown categories, bad difficulty values. Run `node test/cards.test.js`. |
| `.github/workflows/ci.yml` | Automated green gate: runs on every push to the repo. Mirrors the local gate. |
| `PLAN.md` | Living roadmap: persistent phases (P0–P7+), strict sequence, one commit per phase. Read before replanning. |
| `DECISIONS_LOG.md` | Structural decisions: branch base, localStorage schema, persist vs. session state, Design Compiler gate, Constitution guardrails. |
| `LEARNINGS.md` | 12 durable lessons: green gate, localStorage additive-only, textContent-only, decoy quality, fresh authoring over stale cherry-picks, Design Compiler, session vs. persistent state, CI as habit. |
| `PROJECT_STATE.md` | Current snapshot: branch topology, phase status, next actions. Updated after each major phase. |

---

## The Green Gate (Critical — No `tsc`, No `npm run typecheck`)

This project has zero dependencies and no TypeScript. So there's **no `tsc`**. Instead, the green gate is the safety net:

```bash
# 1. Validate cards.js syntax
node --check cards.js

# 2. Extract and validate inline <script> (lines drift as HTML grows — re-derive every step)
open=$(grep -n '<script>' index.html | head -1 | cut -d: -f1)
close=$(grep -n '</script>' index.html | tail -1 | cut -d: -f1)
sed -n "$((open+1)),$((close-1))p" index.html | node --check -

# 2b. Validate the pre-paint theme script in <head> (light mode's no-FOUC hook)
open=$(grep -n '<script id="theme-boot">' index.html | cut -d: -f1)
close=$(awk -v s="$open" 'NR>s && /<\/script>/ {print NR; exit}' index.html)
sed -n "$((open+1)),$((close-1))p" index.html | node --check -

# 3. Run the card validator
node test/cards.test.js

# 4. Browser smoke test (start a local server, then play through):
#    - Start the game (you see a green title bar)
#    - See 4 colored dots (the answer options)
#    - Click one; observe correct/wrong feedback
#    - Press 'C' to cycle category
#    - Press 'L' to switch to Learn mode
#    - Check localStorage: open DevTools → Application → Storage → Local Storage
#      You should see 'gc.v1' with keys: hi, category, cardStats, mode
```

**Run the green gate after every change.** It's cheap, it's fast, and it's the only safety net we have.

---

## Design Tokens (Design to These — No Raw Hex)

**Colors:**
- `--neon-pink` `#e6237f` · `--neon-cyan` `#1ed4e6` · `--neon-purp` `#9d4edd` · `--neon-blue` `#3550e6`
- `--pac` `#ffe600` (Pac-Man yellow) · `--gold` `#ffd700` (lifeline pip)
- `--soft` `#f1e6ff` (body text, off-white for eye comfort) · `--dim` `#b59dde`
- `--dot-1` through `--dot-4` (the 4 answer choices)
- `--ghost-red`, `--ghost-pink`, `--ghost-cyan`, `--ghost-orange`
- `--bg-deep` `#0a0118` (deep purple-black, easier than pure black) · `--bg-mid` `#1a0a3a`

**Shadows & Glow:**
- `--shadow-sm` `0 2px 6px rgba(0,0,0,0.55)` · `--shadow-md` `0 0 18px rgba(255,110,199,0.55)` · `--shadow-glow` `0 0 28px var(--neon-cyan)`

**Fonts** (loaded from Google Fonts, line ~35; tokens at `:root`):
- `--font-ui` → `Inter` (UI, headings, titles, prose) — falls back to `system-ui`, `-apple-system`, `Segoe UI`, sans-serif
- `--font-mono` → `JetBrains Mono` (command / code text) — falls back to `ui-monospace`, `SFMono-Regular`, `Menlo`, monospace

> Note: the color tokens listed above are from the pre-redesign synthwave palette and are stale after the "terminal, not arcade" redesign. The live palette is a calm dark terminal theme keyed off `--accent` (`#3DD8C4`). Treat `:root` in `index.html` as the source of truth, not this list, until it's refreshed.

### Two themes — how to add a colour without breaking one of them

There are now **two** token blocks in `:root`: the dark default, and `:root[data-theme="light"]`
("Platinum"). A `<head>` script resolves `gc.v1.theme` (`system` | `light` | `dark`) and stamps
`data-theme` on `<html>` **before first paint**, so there is no flash. The showcase factory can
force a theme by setting that attribute directly.

Rules when touching colour:

1. **Never write a raw colour in a rule.** Every live rule reads a token; the only literals in the
   sheet are the token definitions themselves. A census script will otherwise find you.
2. **Alpha-composed colours use the channel triplets** — `rgba(var(--accent-rgb), .35)`, never
   `rgba(61,216,196,.35)`. The alphas are per-use design decisions; the channels are shared, so
   one theme swap re-keys everything.
3. **If you add a token, add it to BOTH blocks.** A token defined only in `:root` silently keeps
   its dark value on light — which is exactly how washed-out light modes happen.
4. **Watch for tokens doing two jobs.** `--accent-quiet` (pale fill) and `--accent-deep` (gradient
   shadow end) share a value in dark and must diverge in light. `--ink-on-accent` is *not* the page
   surface. `--overlay-rgb` is white on dark and **black** on light.
5. **`--phantom-eye` stays white in both** — it sits on the teal body, not on the page.
6. **Glows are redesigned, not reused** — a 16px bloom on a light surface is a smudge; light uses
   tight rings at the same loudness rank.

Verify with the harness in `design-reviews/light-mode/tools/`:
`contrast.mjs` (47 measured pairings), `edges.mjs` (theme resolution + the edges),
`fouc.mjs` (screencast proof of no flash), `capture.mjs` (the byte-identity gate).

**A11y Baseline:**
- `#a11y-announcer` (an `aria-live="polite"` div that announces card prompts, feedback, game state)
- `aria-labels` on all interactive buttons (category toggles, mode switches, action buttons)
- `aria-pressed` on toggle buttons (e.g., category filter chips)
- `:focus-visible` rings on all keyboard-focusable elements
- `prefers-reduced-motion` media query honored (disables animations)

Never add a color, font, shadow, or spacing value that's not in this list. If you need one, add it to `:root` first.

---

## Load-Bearing Gotchas (Read These or Debug Later)

### 1. **All Card-Derived Text via `textContent`, NEVER `innerHTML`**

Any text from the card object (`card.prompt`, `card.hint`, `card.explain`, `card.answer`, `card.decoys`) **must** go through `textContent`, never `innerHTML`. This includes the `renderExplain()` function.

Why: Card data is authored by maintainers (trusted at write-time), but using `innerHTML` opens a future injection vector if the code is refactored or if external content ever gets added (e.g., user-written cards). Defense-in-depth.

```javascript
// Good:
DOM.promptText.textContent = card.prompt;

// Bad (never):
DOM.promptText.innerHTML = card.prompt;

// If you need HTML formatting (like <br>):
const div = document.createElement('div');
const line1 = document.createElement('div');
line1.textContent = firstLine;
const line2 = document.createElement('div');
line2.textContent = secondLine;
div.appendChild(line1);
div.appendChild(line2);
```

### 2. **localStorage Stays Additive at `gc.v1` — Never Rename Keys**

The persist schema is `localStorage['gc.v1']` with these keys:
```javascript
{
  hi: number,               // high score / best (NOT a player name — despite the field name)
  category: string,        // 'all', 'claude', 'mac', 'git', etc.
  cardStats: { [cardId]: { mastered: bool, difficulty: string } },
  mode: 'arcade' | 'learn', // current mode
  // ... new fields added as needed
}
```

**Hard rule:** Never rename `hi`, `category`, `cardStats`, or `mode`. Users have saved games in localStorage. If you rename a key, their save data vanishes.

**To add a new field:** Add it to the default object in the load function:
```javascript
const loadPersist = () => {
  const saved = localStorage.getItem('gc.v1');
  return saved ? JSON.parse(saved) : {};
};

const state = Object.assign({
  // defaults
  hi: 0,
  category: 'all',
  cardStats: {},
  mode: 'arcade',
  // new field:
  myNewField: defaultValue,
}, loadPersist());
```

Old saves won't have `myNewField` — the default fills it in automatically.

**If a rename is ever forced:** bump the version to `gc.v2` and write a migration:
```javascript
const saved = localStorage.getItem('gc.v2') 
  || migrateFromV1(localStorage.getItem('gc.v1'));
```

### 3. **Decoys Must Never Be a Real Alias**

A decoy must never be a real alias, abbreviation, or alternate spelling of the correct answer. Example of a bug that shipped: `claude -r` and `claude -p` were decoys for `claude run` and `claude profile` — but `-r` and `-p` are real aliases. A player who knew the aliases could guess wrong and still be correct.

When authoring decoys, ask: "Is this decoy a real alias or documented shortcut?" Check the official docs (Claude Code, Mac terminal, Git). Prefer decoys that are plausibly wrong but not officially valid.

### 4. **Lifeline State Is Session-Only, Not Persisted**

The 50/50 Lifeline feature tracks `state.lifelinesLeft` in memory. It resets to 3 at the start of each game (`startGame()`), **not on app open**. It is **never saved to localStorage**.

Why: Lifelines are a meta-game mechanic to balance a single session. Persisting them would let a player hoard lifelines across sessions, breaking the difficulty curve. Session-only keeps the game fair.

**Extend this pattern:** Any new in-game state that resets between games should be session-only. Only persist data that survives an app restart (high score, category preference, card statistics).

### 5. **Session State vs. Persistent State**

Keep the mental model clean:

**Session-only** (reset on each game start):
- `state.current` (current card)
- `state.score`, `state.livesLeft`, `state.lifelinesLeft`
- `state.missedThisRun` (Set of card IDs missed in this session)
- Game loop variables

**Persistent** (survive app close/reopen, stored in `gc.v1`):
- `hi` (high score)
- `category` (preferred filter)
- `mode` ('arcade' or 'learn')
- `cardStats` (per-card mastered flag, difficulty override)

**Code pattern:**
```javascript
// Session state — initialized in startGame():
state.current = pickCard();
state.livesLeft = 3;
state.lifelinesLeft = 3;
state.missedThisRun = new Set();

// Persistent state — initialized from localStorage, saved on close:
state.hi = loadPersist().hi || 0;
state.category = loadPersist().category || 'all';
```

---

## How to Add a Card

1. Open `cards.js` in any text editor.
2. Find the end of the last card object (before the closing `];`).
3. Copy the entire `{ id: "...", category: "...", ... }` block.
4. Paste it on a new line and change these fields:
   - `id`: unique short string, no spaces (e.g., `"git-status"`)
   - `category`: one of `"claude"`, `"mac"`, `"git"`
   - `difficulty`: (optional) `"easy"`, `"medium"`, or `"hard"` (missing → `"medium"`)
   - `prompt`: the question shown above the Phantom
   - `answer`: the CORRECT command (must match one of the 4 dots)
   - `decoys`: 3 wrong-but-plausible commands (none should be real aliases of the answer)
   - `hint`: short memory aid, shown after wrong answer
   - `explain`: (optional) one or two sentences on WHY, shown in Learn Mode reveal

5. Save the file.
6. Refresh the game tab (if it's open).
7. Run the green gate to verify:
   ```bash
   node test/cards.test.js   # should pass
   ```

**Example:**
```javascript
{ id: "git-status", category: "git", difficulty: "easy",
  prompt: "Show which files you've changed in the current git repo",
  answer: "git status",
  decoys: ["git diff", "git log", "git changes"],
  hint: "Same word your laundry app uses.",
  explain: "git status lists all changed files — modified, added, deleted. It's your first check before committing." }
```

---

## The C-Cycle and Category Buttons

**In Arcade & Learn:**
- Press `C` (or click a button) to cycle through categories: `all` → `claude` → `mac` → `git` → `all` …
- The bottom bar shows category buttons: `ALL`, `CLAUDE`, `TERMINAL`, `GIT`. Clicked buttons toggle the current category and stay visually pressed.
- The card deck automatically filters: if you're in `claude` mode, only Claude Code cards are pickable.

**Code pattern:**
```javascript
const CATEGORY_LABELS = {
  'all': 'ALL',
  'claude': 'CLAUDE',
  'mac': 'TERMINAL',
  'git': 'GIT',
};

const cycleCategory = () => {
  const order = ['all', 'claude', 'mac', 'git'];
  const idx = order.indexOf(state.category);
  state.category = order[(idx + 1) % order.length];
  savePersist(); // persist the new category preference
  renderUI();
};
```

When you add a new category (e.g., `"python"`), update three places:
1. `cards.js`: cards with `category: "python"`
2. `index.html`: `CATEGORY_LABELS['python'] = 'PYTHON'` and add a `<button>` in the `#bar`
3. `index.html`: add `'python'` to the `order` array in the C-cycle handler
4. `test/cards.test.js`: add `"python"` to `VALID_CATEGORIES`

---

## Governance & Constitution

**`main` is Sky's hand only** (Constitution Article 1). Agents write all work to a feature branch. Sky merges to `main` with `git merge --ff-only <branch>` — never agent-delegated, even with in-session approval (one scoped override on 2026-06-03 was explicit and one-off; future merges default to this rule).

**Design Compiler gate** (Constitution Article 2.4): UI changes (buttons, panels, colors, motion) must pass a 7-layer compile gate before marking "DONE":
1. Tokenization — use design tokens, not raw hex
2. Accessibility Parity — contrast, screen-reader text, focus rings
3. Component Consistency — reuse existing `.btn`, `.card`, etc.
4. Visual Entropy — alignment, spacing, weight
5. Luxury UI Score — polish, motion, delight
6. Regression Safety — no unintended side effects
7. Compile Decision — COMMIT (ship), POLISH (refine), BLOCK (rework), or ESCALATE

If your phase touches the UI, route it to Dani (design) for the gate. Don't mark UI DONE without Dani's COMMIT decision.

**PLAN.md is the living roadmap.** Phases are **strictly sequential** — never parallel, because every phase edits the same 2 files (`index.html`, `cards.js`). Read PLAN.md before starting a new phase. Update it when your phase is done: mark it **DONE**, move the next phase to **NEXT**.

**Never commit credentials or secrets.** All sensitive data lives in GitHub Secrets or environment variables, never in files.

---

## Testing & Quality

### Local Green Gate
Run after every change:
```bash
node --check cards.js
open=$(grep -n '<script>' index.html | head -1 | cut -d: -f1)
close=$(grep -n '</script>' index.html | tail -1 | cut -d: -f1)
sed -n "$((open+1)),$((close-1))p" index.html | node --check -
node test/cards.test.js
# then: open index.html, play through, check localStorage
```

### CI Workflow
`.github/workflows/ci.yml` runs on every push and mirrors the local gate. If CI fails, fix the root cause (not the test) before pushing.

### Smoke Test Checklist
- [ ] Game starts with a title bar (no console errors)
- [ ] 4 colored dots appear
- [ ] Capture the correct token → it's chomped, score increases
- [ ] Click a wrong dot → a ghost flashes, life decreases
- [ ] Press `C` → category cycles, deck filters
- [ ] Press `L` → switch to Learn mode
- [ ] In Learn mode, press `ANSWER` → reveal, press `EXPLAIN` → show explanation
- [ ] localStorage `gc.v1` contains `hi`, `category`, `cardStats`, `mode`

---

## Conventions

### Session Flow
1. **Initialize:** Title screen loads; saved best score (`hi`), category, and mode load from `gc.v1`
2. **Pick category:** Press `C` or click button → stored in `category`, persisted
3. **Pick mode:** Click `ARCADE` or `LEARN` → stored in `mode`, persisted
4. **Play:** Game loop picks random cards, player answers
5. **Game over (Arcade only):** Show final score, list missed cards (if any)
6. **Learn mode:** After each card, reveal explanation; player decides next/retry
7. **Close app:** `gc.v1` stays in localStorage; player's state is preserved

### Code Style
- No `any` types (but the codebase is untyped, so this is N/A).
- Inline `StyleSheet` equivalents (just use classes, not inline styles — design tokens go in `:root`).
- Use `const` for functions and state; `let` for loop counters.
- Use `textContent` for all user-facing text; `innerHTML` never.
- Reusable DOM elements: `const DOM = { promptText: ..., answerDots: ..., announcer: ... }`.

### Persist Pattern
```javascript
const savePersist = () => {
  const persist = {
    hi: state.hi,
    category: state.category,
    cardStats: state.cardStats,
    mode: state.mode,
  };
  localStorage.setItem('gc.v1', JSON.stringify(persist));
};

const loadPersist = () => {
  const saved = localStorage.getItem('gc.v1');
  return saved ? JSON.parse(saved) : {};
};
```

---

## QA Reports & Governance

All findings, proposals, and summaries live in `qa-reports/` (per-project convention, not central). Filename format:

- `2026-06-03_RoleName_Report.md` — direct role invocation
- `qa-reports/INDEX.md` — triage guide; read this first when scanning QA
- `cycle-2026-06-03-morgan.md` — orchestrator cycle output
- `2026-06-03_DesignCompile_p6-git-button.md` — Design Compiler gate result (PASS/BLOCK/POLISH/ESCALATE)

Morgan reads these files and routes findings to the team or Sky. Don't flood Sky directly; surface everything to your qa-report, let Morgan drive priorities.

---

## Recent Status

- **Live:** The "terminal, not arcade" Ghost Code redesign is **shipped** at https://ghostcode.skypistudio.com (`origin/main` = `1e6b963`, 2026-09-02). 56 cards, calm dark terminal palette, green gate passing.
- **Identity:** Calm, modern terminal-command trainer. Phantom mascot, `--accent` (`#3DD8C4`) keyed theme, Inter + JetBrains Mono. No more arcade/synthwave framing.
- **Shipped since the last sync:** the identity/attribution pass has merged (start-screen "Built by Sky Halisky" + Source credit, HUD relabel SCORE/BEST, `@supports` wordmark fallback) — no longer "in flight." Also shipped: the light-mode "Platinum" theme (dark-value-preserved tokenization, palette selection, mechanism + edge cases) with a `T` keyboard shortcut and a title-screen `THEME` button, plus a fix for that button being unreachable before a game started. Nothing is currently in flight for Ghost Code.
- **Merge rule:** `main` is Sky's hand only — agents branch, Sky merges.

---

## When the User (Sky) Asks for Changes

- Sky is a beginner coder learning by doing. Explain what you're doing at key moments — terse, in plain language, not jargon.
- Prefer editing existing files over adding new ones.
- Always run the green gate before declaring something done.
- Never merge to `main` yourself; write to a feature branch and surface to Morgan/Sky.
- If a change requires new design tokens, add them to `:root` first (don't invent raw hex).

---

## Contacts

- **Owner:** Sky (skylerhalisky@gmail.com)
- **Project Coordinator:** Morgan (routes all findings and decisions)
- **Design:** Dani (owns Design Compiler gate for UI changes)
- **QA:** Gary (test infra, green gate automation)
- **DevOps:** Rory (CI/CD, branch merges under Sky's gate)

---

## Quick Links

- **Play it (live):** https://ghostcode.skypistudio.com
- **Run locally:** `cd ~/Games/pacman-code-trainer && python3 -m http.server 8000`, then open `http://localhost:8000`
- **Repo (local):** `/Users/skypie/Games/pacman-code-trainer` · **Source:** https://github.com/Skypie99/ghost-code
- **Read first:** PLAN.md (living roadmap), DECISIONS_LOG.md (structural decisions), LEARNINGS.md (durable lessons)
