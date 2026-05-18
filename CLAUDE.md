# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Time's Down** is a French-language, local (single-device, pass-the-phone) party word game — a "Time's Up" clone. The entire app is one self-contained `index.html` (inline CSS + JS, no build step, no framework) plus a service worker. UI strings are in French; keep new user-facing text in French.

## Running & testing

There is no build, lint, or test tooling. To develop, serve the directory over HTTP and open it in a browser — the service worker and PWA manifest do **not** work from `file://`:

```
python3 -m http.server 8000   # then open http://localhost:8000
```

Testing is manual in a browser. When testing changes that touch caching/offline or persistence, note that:
- The service worker caches the app shell. After editing `sw.js` or `index.html`, bump `CACHE` in `sw.js` (currently `"timesdown-v1"`) or hard-reload / clear the SW, otherwise you'll see stale code.
- Game state persists in `localStorage` under `timesdown_state_v1`. Clear it (or use the in-app "Nouvelle partie") to test the setup flow from scratch.

## Architecture

### Single global state + screen router
All app state lives in one object `S` (shape defined by `freshState()`). The app is a state machine: `S.screen` names the current screen, and `render()` is a switch over `S.screen` that wipes `#app` and rebuilds it. Screen flow:

`setup → teams → wordEntryForm → turnReady → countdown → playing → turnRecap → (loop back to turnReady / next round, or gameOver)`

Mutate `S`, then call `save()` and `render()` (or a partial updater) — never manipulate the DOM as the source of truth.

### Rendering convention
DOM is built with the `el(tag, props, ...children)` helper (`class`, `html`, `on*` event props are special-cased). `render()` does a full teardown/rebuild. For hot paths during a turn, partial updaters mutate existing nodes instead of re-rendering: `updatePlaying()` (word/counters) and `updateTimerUI()` (timer ring/number). Icons are Lucide; after building DOM call `lucide.createIcons()` — `render()` already does this, and the boot code re-draws icons once the deferred Lucide CDN script loads.

### Persistence & migration
`save()`/`load()`/`clearSave()` wrap `localStorage` (key `STORAGE_KEY`). `load()` contains **forward-migration logic** for older saved shapes (e.g. converting numeric `rounds` to objects, backfilling `deadline`/`stats`). When you change the shape of `S`, add migration handling in `load()` so existing players' saved games don't break, and bump the storage key only as a last resort.

### Turn mechanics (the tricky part)
- Each game has rounds (default 3: description libre / un seul mot / mime) played over the **same** set of player-entered words; `S.roundRemaining` is the pool for the current round.
- A turn shuffles `roundRemaining` into `S.pile`, serving one `currentWordId` at a time. `actionFound` / `actionPass` advance the pile; `actionUndo` pops a snapshot.
- **Undo** works via a snapshot stack (`S.snapshots`, JSON strings pushed by `snapshot()`) capturing pile/current/remaining/found. Score is recomputed from the snapshot's found-count delta, not stored separately.
- **Drift-free timer**: `startTimer()` stores an absolute `S.deadline` (epoch ms) and derives remaining time from `Date.now()`, so a backgrounded tab stays accurate.
- **Interrupted turns**: if the app reloads mid-turn (`renderResume`), `rollbackTurn()` refunds points scored that turn and returns those words to `roundRemaining`, so no points are kept for an unfinished turn.
- **Rotation**: `continueAfterRecap()` is the single place that advances `descPtr` (next describer within the team) and `teamPos` (next team), applies contested-word removals (`S._removed`), credits per-player/per-round stats, and decides round-over / game-over / tiebreaker.
- Tiebreaker: `startTiebreaker()` clones the tied teams into a sudden-death single round, stashing originals in `S._origTeams`/`S._origRounds`.

### Side-effect subsystems (all degrade gracefully, no asset files)
- **Audio**: Web Audio oscillator tones via `tone()`; `unlockAudio()` must run from a user gesture (the "JOUER" tap) to satisfy mobile autoplay rules.
- **Wake Lock / vibration**: `requestWakeLock()` keeps the screen on during a turn (re-acquired on `visibilitychange`); `vibrate()` for haptics. All wrapped in try/catch — assume APIs may be absent.
- **PWA without assets**: `setupPWA()` generates the manifest (as a Blob URL) and the app icon (SVG data URI) at runtime — there are intentionally no icon/manifest files on disk. `sw.js` is network-first for navigation (fresh when online, cached fallback offline) and stale-while-revalidate for other GETs.

## Gotchas

- Always render user-controlled strings (player names, words) through `escapeHtml()` when injecting via the `html` prop; prefer text children otherwise.
- The inline script runs before the deferred Lucide CDN load, so first paint has no icons until the boot-time re-draw — don't "fix" this with a synchronous dependency on `lucide`.
- Don't introduce a build step, framework, or external runtime dependency without explicit instruction — the zero-tooling, single-file, offline-first design is a core constraint of this project.
