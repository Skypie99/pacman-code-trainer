# Ghost Code — Roadmap (PLAN.md)

> Living roadmap for build bursts. Each burst continues here instead of re-deciding.
> Owner: Morgan (planning) → roles execute. **Never merge to `main` — Sky's gate.**
> Last reconciled: **2026-06-02 (updated)** (burst `burst/pacman-2026-06-02`).

## What this project is
A vanilla HTML/JS **terminal-command trainer** — Ghost Code, the Phantom mascot, not
Pac-Man (rebranded 2026-06-04 over trademark risk; see DECISIONS_LOG.md) — for
memorizing Claude Code + macOS terminal + Git commands. Single file `index.html`
(~3.3k lines, inline CSS+JS) + `cards.js` (`window.CARDS`). **Zero deps, no build, no
framework, no TypeScript.**

## Green gate (the "type check" equivalent — run after every step)
```
node --check cards.js                                    # exit 0
open=$(grep -n '<script>' index.html | head -1 | cut -d: -f1)
close=$(grep -n '</script>' index.html | tail -1 | cut -d: -f1)
sed -n "$((open+1)),$((close-1))p" index.html | node --check -   # exit 0
# then: serve :8080, 0 console errors, smoke (start · 4 dots · correct/wrong · C · L · localStorage)
```
Data layer = `localStorage['gc.v1']` only (one-time migration from legacy `pmct.v1` exists). **No DB.** Keep persist changes
additive (new keys on the `Object.assign({defaults}, loadPersist())` default) — never rename
`hi/category/cardStats/mode`; bump to `gc.v2` only with a migration if a rename is ever forced.

## Design tokens (design to these — no new colors)
`--accent #3DD8C4` (single teal accent) · `--surface-base` · `--text-primary` / `--text-secondary` ·
`--border-subtle` · `--dot-1..4` (answer choices) — calm dark "terminal not arcade" palette
(treat `:root` in `index.html` as source of truth). Fonts: `Inter` (UI), `JetBrains Mono` (command/code).
A11y baseline: `#a11y-announcer` aria-live · aria-labels · `aria-pressed` toggles ·
`:focus-visible` rings · `prefers-reduced-motion`.

---

## Phases

| # | Phase | Roles | Status |
|---|---|---|---|
| P0 | Foundation & reconciliation | Morgan, Gary | **DONE 2026-06-02** |
| P1 | Learning Mode Design Compiler gate + a11y fold-in | Dani, Alex, Shamus | **DONE 2026-06-02** |
| P2 | Card-level explanations (Riley P3) | Quinn, Dani, Dana, Shamus, QA, Gary | **DONE 2026-06-02** |
| P3 | Difficulty tags + content quality | Dani, Dana, Shamus, QA, Gary | **DONE 2026-06-02** |
| P4 | 50/50 Lifeline (H key, 3 per game) | Dani, Shamus, QA | **DONE 2026-06-02** |
| P5 | Card-validation test + CI adoption (infra) | Gary, Rory | **DONE 2026-06-02** |
| P6 | GIT command cards fold-in (~16 cards + GIT category button) | Shamus, QA, Gary | **DONE 2026-06-02 — Design Compiler pending** |
| P7 | Arcade game-over review + card-prompt announcements | Shamus, Alex, QA | **DONE 2026-06-03** |
| P8 | Mobile-native + WCAG 2.2 AA + polish (Sky-direct) | Sky/Claude | **DONE 2026-06-05** |

Strict sequence — never parallel (every phase edits the same 2 files). One commit per phase.

> **Storage key note:** the live data layer is `localStorage['gc.v1']` (one-time migration from legacy `pmct.v1`). Older "pmct.v1" mentions below are historical — the rule (never rename keys; extend additively) is unchanged.

### P1 — Learning Mode Design Compiler gate + a11y fold-in
Learning Mode is built + smoke-tested but never passed the 7-layer Design Compiler (its report
is absent → can't be marked UI DONE). Run the gate; apply predicted polish:
- **L1 token:** move `#learn-panel` inline `style=` colors/sizes → modifier classes.
- **L2 focus:** focus `#lp-retry` on retry inject, `#lp-next` on reveal (keyboard users currently stuck on the disabled dot).
- Fold in Taylor's 3 category-chip `aria-label`s (keep existing `aria-pressed`).
- Output: `qa-reports/2026-06-02_DesignCompile_pm-learning-mode.md`.

### P2 — Card-level explanations
Optional `explain` string on cards; fallback `card.explain || card.hint || ''`. `renderExplain()`
via **`textContent` only**. Surface in Learn reveal/correct (convert those branches off
`innerHTML`) + Arcade hint line. Reuses pre-built `.lp-explain` CSS. No schema bump.

### P3 — Difficulty tags + content quality
Optional `difficulty: easy|medium|hard` (missing → medium). Token-only badge (easy→cyan,
medium→gold, hard→pink) with text label + `aria-label` (not color-only). P3 owns the prompt-tag
`ternary → tagLabels` refactor. Optional `pickCard` weighting `{easy:1.2,medium:1,hard:0.7}`.
Rewrite oblique hints.

### P4 — 50/50 Lifeline ✅ DONE
H key eliminates 2 wrong dots. 3 lifelines per game (both modes). Gold ⚡ pip HUD indicator.
`state.lifelinesLeft` (session only, no persist). Idempotent per card; dots restored on `nextCard`.

### P5 — Card-validation test + CI adoption (infra) ✅ DONE
`test/cards.test.js` (zero-dep, validates difficulty/explain/decoys/answer-in-decoys).
`.github/workflows/ci.yml` mirrors the local green gate (node --check + validator + html smoke).

---

### P6 — GIT command cards fold-in  ·  NEXT  ·  roles: Shamus, QA, Gary
~16 git cards written from scratch (the stale `feat/auto-2026-05-25-shamus-git-*` branch has
the content but a conflicting old base — write fresh, do NOT cherry-pick the UI commit which
touches line 960 in the old tree).

**cards.js changes:**
- Add ~16 `{ id: "git-*", category: "git", difficulty, prompt, answer, decoys, hint }` cards
  (commands: init, clone, status, add, commit, push, pull, log, diff, branch, checkout, merge,
  stash, reset HEAD, rm cached). Tag each with difficulty.
- Update `VALID_CATEGORIES` in `test/cards.test.js` to include `"git"`.

**index.html changes:**
- `CATEGORY_LABELS`: add `git: 'GIT'`.
- `activeDeck()`: already category-agnostic — no change needed.
- Bottom bar: add `<button class="btn" data-cat="git" aria-pressed="false" aria-label="Show Git commands only">GIT</button>` after TERMINAL.
- C-key cycle: update `const order = ['all','claude','mac','git']`.
- No new CSS needed (`.btn` styling is already shared).

**Persist:** `category` already stores arbitrary strings; old saves default to `'all'` which
includes GIT cards automatically. Additive, no schema bump.

**Content note:** add `explain` to the trickier GIT cards (e.g., stash vs reset, merge vs rebase).

---

### P7 — Arcade game-over review + card-prompt announcements (a11y)  ·  roles: Shamus, Alex, QA

**Part A — Arcade game-over missed-cards review:**
- `state.missedThisRun = new Set()` — reset in `startGame()` alongside `learnMastered`.
- In `answer()` wrong branch (arcade path ~line 1195): `state.missedThisRun.add(state.current.id)`.
- In `gameOver()`: if `state.missedThisRun.size > 0`, build a scrollable `<div>` inside
  `#gameover` listing each missed card's prompt + explain (via `textContent`, never innerHTML).
  Use `.lp-explain` CSS class — already styled and correct contrast. Limited to the last 5–8
  misses to keep the screen readable.
- No new persist keys needed.

**Part B — Card-prompt screen-reader announcement:**
- In `renderCard()`, after setting `DOM.promptText.textContent`, add:
  `if (DOM.announcer) DOM.announcer.textContent = \`${CATEGORY_LABELS[card.category] || card.category}: ${card.prompt}\`;`
- This closes the pre-existing cross-mode a11y gap noted in the Design Compiler report
  (prompts are currently never announced; only hints/feedback are).

Both parts are small, independent, and touch `index.html` only.

---

### P8 — Mobile-native + WCAG 2.2 AA + polish  ·  DONE 2026-06-05  ·  branch `feat/ghost-mobile-a11y-2026-06-05`

Sky-direct pass (mirrors the Prompt Library run). 6 per-area commits, `index.html` only:
1. **Touch foundation** — `viewport-fit=cover`, `touch-action`/`overscroll`/tap-highlight, safe-area insets; `#arena{touch-action:none}`; panels stay touch-scrollable.
2. **Responsive scale** — arena/token/prompt fluid (`clamp/min`, desktop unchanged); portrait layout fixes the mode-toggle↔HUD overlap, centres + scales the compass diamond so N/E/S/W never clip, re-anchors the learn-mode panels.
3. **Swipe-to-answer** — swipe up/right/down/left → N/E/S/W via existing `answer()`; tap still works; `:active` feedback + reduced-motion-aware haptics.
4. **Control parity** — 50/50 lifeline button (was keyboard-only `H`); touch-aware title hint.
5. **WCAG 2.2 AA** — 1.4.1 ✓/✗ glyph; 2.3.1 flash capped to one bloom; 2.5.8 targets ≥24px; pause announced via live region.
6. **Polish + docs** — button `:active` press feedback; this report + PLAN/PROJECT_STATE/LEARNINGS.

**Reality clarified:** Ghost Code is a 4-token directional *quiz*, not a moving maze. Full report: `qa-reports/2026-06-05_GhostCode_Mobile_A11y_Polish.md`. Open follow-up: learn mode hides the question behind the hint panel (pre-existing on desktop too).

---

## Merge queue (advisory — Sky gates `main`; agents delete/merge nothing)
**Ready / Design-PASS:**
- `burst/pacman-2026-06-02` (headline — contains `main` + Learning Mode + this burst's P1–P3/P5).
- CI workflow `dfe352d` (clean new files) — folded into P5 / or merge separately.

**Needs a decision:**
- **GIT-cards feature** (`feat/auto-2026-05-25-shamus-git-*`, ~16 cards + UI) — stale base, UI wiring conflicts with P3's ternary. **Deferred** to a follow-up burst by default; say the word to fold into P3.
- `Taylor/a11y/category-aria-labels-2026-05-27` — folded into P1; prune after burst merges.

**Already absorbed into local `main` (safe to prune):**
- `feat/pacman-premium-polish-2026-05-29`, `feat/pacman-round2-polish-2026-05-29`, `fix/answer-card-contrast-2026-05-29` (0 unique commits vs main).

**Stale `*/auto-*` (old base — prune at discretion):**
- `a11y/auto-2026-05-25`, `cycle/auto-2026-05-23-ui`, `community/auto-2026-05-25-casey-readme`,
  `release/auto-2026-05-25` (keep until its CI commit is taken), the two `test/*` branches
  (cherry-pick the test file only — they revert `index.html` / delete qa-reports wholesale).

**Note:** local `main` is +21 vs `origin/main` (origin still at Initial commit). Review with
`git diff main..burst/pacman-2026-06-02` (local main = clean). Pushing `main` to origin is Sky's call.
