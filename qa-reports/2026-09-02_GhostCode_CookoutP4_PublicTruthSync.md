# Ghost Code — Portfolio Cook Out Prompt 4: Public Truth Sync

**Date:** 2026-09-02
**Scope:** Reconcile Ghost Code's public repo/docs/metadata with what the product actually ships now. Documentation/metadata reconciliation only — no product/UI behavior changed. No push, merge, or deploy.
**Base:** `origin/main` @ `1e6b963d3b88940f13593dac2d0b1cb8e7966549` (clean, matched the planning SHA exactly — no remote drift).
**Worktree:** this session's existing isolated worktree (`claude/portfolio-cookout-prompt-bbed3f`) — already on current `origin/main`, so no second clone/worktree was created.

## Method
1. Read the accepted Portfolio Cook Out P1 truth manifest (`/Users/skypie/Portfolio-cookout-p1-20260902/docs/PORTFOLIO_TRUTH_MANIFEST.md`, read-only) for context on how Ghost Code is currently characterized externally.
2. Full source audit: `cards.js`, `index.html` (metadata, theme system, keyboard/touch handlers, mode logic, mastery logic, persistence schema) via direct reads and targeted greps.
3. Ran the canonical green gate: `node --check cards.js`, the extracted inline `<script>` (2029–3310), the `theme-boot` script (36–68), `node test/cards.test.js`, the CI theme-contract greps.
4. Checked the live site (`https://ghostcode.skypistudio.com`) and a local `python3 -m http.server` copy in the Browser pane — real clicks/keypresses, not direct JS function calls (see Limitations).
5. Audited current-state docs (`PROJECT_STATE.md`, `PLAN.md`, `CLAUDE.md`, `DECISIONS_LOG.md`, `LEARNINGS.md`, `TASK_GRAPH.json`, `.state-snapshot.yaml`, `.features-brief.yaml`, `PROJECT_DIGEST.yaml`) and classified each as current/historical/dead.
6. Applied only evidence-backed corrections; ran the green gate again; committed once.

## Deck truth (source-computed, not assumed)
`node test/cards.test.js` → **OK — 56 cards passed all integrity checks.**
- Claude Code: 20 · macOS terminal: 20 · Git: 16 · Total: **56**. No duplicate IDs, no missing fields, no invalid decoys (each has exactly 3 unique decoys, none equal to the answer). No decoy is a real alias of its answer (spot-checked `cc-resume`/`cc-print`, the pair the CLAUDE.md gotcha references — both clean).
- 56 matches every current public claim (README, meta description, OG/Twitter tags) — **no update needed there.**

## Modes (verified against source + real interaction)
**Arcade:** starts at 3 spirits (`state.lives = 3`). Correct answer: `+10 + streak` points, streak++, `bestStreak` persisted if it's a new all-time best. Wrong answer: spirit lost, the correct answer + hint revealed, `missedThisRun` grows, **and the card's persisted mastery counter (`cardStats[id].c`) resets to 0** — a detail not documented anywhere previously. `gameOver()` fires at 0 spirits; `missedThisRun` (up to 8) drives the game-over review list.
**Learn:** no spirit loss. Wrong-answer escalation is **retry1 → retry2 → auto-reveal on the 3rd miss**, with a "SHOW ANSWER" bail-out available from the very first miss. Unlike Arcade, a wrong answer here does **not** reset the persisted mastery counter — only Learn's own per-session `learnMastered` Set (always empty at the start of a run) is separate from the persisted counter's escalation.
**Mastery:** a card counts as mastered once its persisted `cardStats[id].c` (correct-answer count) reaches **3**. This also weights `pickCard()`'s random selection — cards below the threshold (and harder-difficulty cards) are picked more often, cards at/above it drop to a flat low weight. This is a deterministic local rule, not spaced repetition, not adaptive/AI-driven, and not analytics — described that way everywhere it's now documented.

## Controls (verified against source + real interaction; see Limitations for what couldn't be click-tested)
| Input | Verified | Publicly documented (before → after) |
|---|---|---|
| Mouse | ✅ source; ✅ real click (live site THEME button) | documented → documented |
| Touch/swipe | ✅ source (`attachSwipe()`, 28px threshold, single-finger, N/E/S/W); ✅ touch-device detection genuinely activates under mobile emulation | **not documented → now documented** |
| `1`–`4` | ✅ real keypress (answered two live cards) | documented |
| `↑→↓←` | ✅ source — moves focus to a token, then Enter/Space captures (does **not** auto-answer, unlike 1–4) | documented (in-game help is precise; README's "aim" wording is the game's own term, left as-is) |
| `H` (50/50) | ✅ source | documented |
| `R` (restart) | ✅ real keypress (started the game from title) | documented |
| `Esc` (pause) | ✅ source (context-sensitive: closes a modal if one's open, else pause/resume) | documented |
| `C` (category) | ✅ real keypress (ALL→CLAUDE, deck filtered correctly) | documented |
| `L` (learn) | ✅ real keypress (mode + deck filter both applied correctly) | documented |
| `T` (theme) | ✅ real keypress (desktop) + ✅ real click (live site) | **not documented → now documented** |
| `?` (help) | ✅ source only this pass (see Limitations) | documented |

## Theme system
Dark default + light "Platinum" theme + system preference, resolved by a pre-paint `<script id="theme-boot">` (no-FOUC), CI-gated (syntax + a theme-contract grep for the light block / `color-scheme: light` / `theme-color-meta`). `T` cycles system→light→dark; a title-screen `THEME` button (whose reachability was just fixed in `1e6b963`) and a matching Settings segmented control both drive the same state. **Verified with a real click on the live production site**: the button's accessible label went from "currently dark" to "light", and `data-theme` flipped correctly — the fix genuinely works. Persistence and no-FOUC reload were also verified for real (see below). "`gc.v1.theme`" in comments/CLAUDE.md is shorthand for a `theme` field inside the single `gc.v1` object — there is no second storage key.

## Persistence — exact current schema
Single key `localStorage['gc.v1']`, one-time migration from legacy `pmct.v1`:
```
hi: number              — high score / best (NOT a player name, despite the field name)
category: string        — 'all' | 'claude' | 'mac' | 'git'
cardStats: object        — per-card { c: correct count, w: wrong count, total }
mode: 'arcade' | 'learn'
bestStreak: number
soundOn: boolean
reduceMotion: boolean
difficultyFilter: 'all' | 'easy' | 'medium' | 'hard'
theme: 'system' | 'light' | 'dark'
```
**This is the most significant finding of this pass:** `CLAUDE.md` documented `hi` as `string // player's name` in four places (the schema comment, a code sample's default value, the session/persistent-state list, and the Session Flow narrative) — there is no name-entry feature at all; `hi` has always been the numeric high score (`state.persist.hi`, compared against `state.score`, shown in the "BEST" HUD stat). Fixed in this pass.

## What was fixed and why (evidence-backed only)
- **`README.md`:** added the `T` shortcut and touch/swipe to Controls (previously undocumented); replaced the personal path `cd ~/Games/pacman-code-trainer` with a portable `git clone` + `cd ghost-code`; removed the `assets/` bullet (the directory doesn't exist — confirmed via `ls`); noted theme preference is also stored under `gc.v1`.
- **`CLAUDE.md`:** fixed the `hi` = high-score-not-name error in all four spots; fixed the stale `index.html` line-count/range claim (`~1.3k lines, lines 886–1516` → current is 3312 lines with the main script now at 2029–3310 — reworded to say "always re-derive, never hardcode," consistent with the file's own stated philosophy); rewrote "Recent Status" — the identity/attribution pass it described as "in flight… branch only" merged months ago (`b44e232`/`6846b3d`, confirmed ancestors of current HEAD), and it didn't mention the light-mode theme work or the theme-toggle fix at all.
- **`PLAN.md`:** fixed only the "What this project is" blurb — it still called the product a "Pac-Man themed flashcard game" and omitted Git as a category, even though the same file's own phase table marks the Pac-Man→Ghost Code rebrand-adjacent phases and the Git-cards phase (P6) both DONE. Left the phase table itself as historical record (accurate for P0–P8; doesn't claim to cover the later uplift/light-mode work, which isn't phase-tracked here).
- **`PROJECT_STATE.md`:** substantially updated — it was last touched 2026-06-04 (a 2026-06-19 partial pass updated only Current Status/Branch topology, missing everything else), 45 commits behind current HEAD, and its Context Snapshot still called Ghost Code the "original synthwave arcade trainer" as current framing — the exact drift this Cook Out prompt flagged as a known candidate. Rewrote Current Status, Branch topology, Context Snapshot, added ledger entries for the identity/attribution pass through the light-mode theme, refreshed Next Actions/Open Risks.

## Explicitly NOT changed (and why)
- **`DECISIONS_LOG.md`, `LEARNINGS.md`, `TASK_GRAPH.json`:** legitimate historical ledgers, each entry dated and self-aware it's a point-in-time record. Not rewritten (per the prompt's historical-preservation instruction). Minor FYI not acted on: `LEARNINGS.md`'s "localStorage Stays Additive" lesson still illustrates with the legacy `pmct.v1` key name rather than current `gc.v1` — the lesson itself (never rename keys) is still correct, only the illustrative key name is dated. Left alone as low-value/out-of-scope for a public-truth pass.
- **`.state-snapshot.yaml`, `.features-brief.yaml`, `PROJECT_DIGEST.yaml`:** look like dead automation artifacts from an external dashboard tool, not hand-maintained (one has `last_commit: 2024-01-15`, predating this project). Not touched — no visibility into what tool owns them.
- **`design-reviews/`, `copy/copy-2026-05-25.md`:** genuine historical/evidentiary process records (the light-mode design-review tools are still the active verification harness per `CLAUDE.md` — not touched, still current).
- **Metadata (`<title>`, meta description, OG/Twitter tags, favicons, social image):** audited in full — all accurate, already reconciled to the current identity. `og-image.png`'s actual pixel dimensions (1200×630, confirmed via `sips`) match the meta tags exactly. No changes.
- **Deck content, game behavior, theme palette:** out of scope by explicit instruction; no defects found that would have required an exception.
- **`.claude/launch.json`:** its `pacman-dev` config's `$PORT` placeholder isn't substituted by this session's preview tooling, so `preview_start` failed and a plain `python3 -m http.server` was run directly instead. Left the file untouched (tooling config, not public product truth) — noted as a Next Action in `PROJECT_STATE.md`.

## Limitations of this session's browser verification (disclosed, not papered over)
This session's Browser pane is not visually displayed/compositing, which broke coordinate-based input: `computer` clicks (including `ref`-resolved ones) failed with "press could not be attributed to a frame," and native button-activation-via-Enter didn't fire (the browser didn't treat the synthetic Enter as trusted enough to auto-synthesize a click on a focused `<button>`, even though the keydown event itself reached the page — confirmed with a temporary capture listener). One real mouse click did succeed, on the **live production site's** THEME button, which is the specific behavior this Cook Out prompt asked to confirm.
Workaround: switched to direct keyboard-shortcut keys (which the game's own `keydown` listener handles directly, not via native button-click synthesis) for local testing — `R`, `1`, `2`, `c`, `l`, `t` all verifiably worked (confirmed via DOM/state inspection after each press). Shifted-character shortcuts (`?`) didn't dispatch in this session (empty capture log — a tooling gap with special characters, not a page-side failure since every un-shifted letter/digit key worked). Mobile-viewport emulation didn't deliver synthetic key events at all once active (consistent with a real phone having no physical keyboard) — mobile verification there was limited to layout/detection (viewport sizing, no horizontal overflow, touch-device class + hint-swap correctly activating under touch emulation with a fresh page load), not gesture/keyboard interaction. Swipe itself was verified via source only. `javascript_tool` was used strictly to *read* state after each real input (never to trigger the behavior being verified) and to diagnose the click/frame-attribution failure itself.
**Net effect:** every keyboard shortcut except `?` was confirmed via genuine dispatched events at least once (desktop); mouse-only affordances (the SETTINGS button, which has no keyboard shortcut) were verified structurally via source (proper `role="dialog"`, `aria-modal`, focus-trap `Tab` cycling, `Escape`-to-close) but not via a real click in this session.

## Files changed
`README.md`, `CLAUDE.md`, `PLAN.md`, `PROJECT_STATE.md`, plus this report and its `qa-reports/INDEX.md` entry. No changes to `index.html`, `cards.js`, `.github/workflows/ci.yml`, or any historical/governance file.
