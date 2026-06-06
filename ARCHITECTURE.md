# Moosytype — Architecture & Setup

A quick-read guide to how this repo is structured and how the app works.

## TL;DR

Moosytype is a **typing trainer for ergonomic / split / matrix keyboards** (ZMK, MoErgo, etc.), built to feel like [Monkeytype](https://monkeytype.com). It is a **single-page, zero-dependency, zero-backend web app** — essentially one big `index.html` file that runs entirely in the browser and is deployed for free on GitHub Pages.

Live site: https://moosylog.github.io/moosytype/

## Files (the whole repo)

| File | Size | Purpose |
|------|------|---------|
| `index.html` | ~106 KB / 1609 lines | **The entire app** — HTML markup, CSS, and JavaScript all inline. This is where 99% of the logic lives. |
| `manifest.json` | tiny | PWA manifest (name, icons, theme color `#323437`, standalone display). |
| `sw.js` | 8 lines | Minimal service worker — a pass-through `fetch` handler that exists only to satisfy Chrome's "installable PWA" requirement. Not a real offline cache. |
| `icon-192.png`, `icon-512.png` | — | PWA app icons. |
| `README.md` | — | Marketing-style overview, feature list, supported keyboards, usage instructions. |

There is **no build step, no package.json, no node_modules, no framework**. You open `index.html` and it runs. Deployment = push to `main`, GitHub Pages serves it.

## How `index.html` is laid out

The single file is divided into three regions:

| Lines | Region | Contents |
|-------|--------|----------|
| 1–217 | `<head>` + `<style>` | Meta tags; all CSS. Theme uses Monkeytype-aligned CSS variables (line ~10). Sections: theme vars, globals, overlays, header/nav, focus mode, settings submenu, scoreboard, toasts, typing area, keyboard, stats, modal. |
| 219–460 | `<body>` markup | Static DOM: toast, start overlay, Tab-restart overlay, header (`#mainHeader`) with mode buttons (words/news/custom) and selects, settings menu (`#settingsMenu`), typing area, SVG keyboard container, stats/scoreboard, custom-text modal. |
| 461–1607 | `<script>` | All app logic (vanilla JS, no modules). Organized into 5 commented banner sections — see below. |

## JavaScript sections (inside the `<script>`)

The script is split by big comment banners:

### 1. Hardware Geometry & Layout Data (lines 462–499)
- `APP_VERSION` (currently `"1.3.0"`) — bumping this wipes `localStorage` on next load (see `window.onload`).
- `commonWords` — fallback top-100 English words, replaced at runtime by `loadDictionary()` which fetches Monkeytype's real 200-word english.json from GitHub.
- **`HARDWARE`** — the core data structure. One entry per supported keyboard (`ansi60`, `glove80`, `go60`, `corne`, `voyager`). Each has:
  - `name` — display name
  - `geo` — array of per-key `{x, y, w, r, rx, ry}` positions/rotations used to draw the SVG.
  - `pos` — array of ZMK matrix position IDs (e.g. `LH_C4R2`, `K07`) parallel to `geo`.
  - `layouts` — `qwerty` / `colemak-dh` / `dvorak` arrays of key labels parallel to `pos`.
- `ZMK_EXCEPTIONS` — maps ZMK keycode names (`N1`, `BSPC`, `FSLH`, `SQT`…) to display characters.

### 2. State, Cache, DOM (lines 500–549)
- **`config`** — user preferences (hardware, source/target layout, mode, word count, timed, toggles for keyboard display/homerow/keypress/hints/mirror/sound/debug, plus any uploaded/dragged custom JSON maps). Persisted to `localStorage` as `moosytype_cfg`.
- **`state`** — live session data (typed text, sample text, current/source maps, test progress, mistakes, streaks, per-finger hit stats, WPM snapshots, timers).
- **`DOM`** + `cacheDOM()` — grabs all element references once into a flat object.

### 3. Storage & Configuration (lines 551–~1160)
The largest section — the actual engine. Notable functions:
- `loadConfig` / `saveConfig` / `setPref` — persistence.
- `parseZMKJson` / `extractZMKLabel` — the **"LayerForge AST engine"**: deep-parses uploaded ZMK / MoErgo `.json` layouts, strips modifiers/behaviors (`&mt`, `&lt`, macros) down to base characters.
- `buildMap` / `getMirroredMap` / `isHomerow` — turn hardware + layout choice into a position→character map.
- `initKeyboard` / `renderKeyboard` / `getFingerName` — draw and update the live SVG keyboard.
- `handleDragStart` / `updateGhostPos` — drag-and-drop key remapping (creates a `custom_dragged` layout on the fly).
- `setupNewTest` / `generateWords` / `fetchNews` — produce practice text (random words / live RSS from NPR & NYT / custom).
- `highlightExpected` / `updateTextUI` — caret + next-key visual feedback.
- `endTest` / `checkTestEnd` — finish a run and compute results.
- `playErrorSound` — Web Audio API synthesizer mimicking a mechanical switch (no audio files).

### 4. Practice, History & Stats (lines 1162–1355)
- `loadHistory` / `saveResult` — store results in `localStorage` (`moosy_history`).
- `practiceMistakes` / `getRecentMistakeChars` — generate a custom drill targeting your weakest characters.
- `renderBestScores` / `renderHistorySummary` / `renderFingerStats` / `renderSparkline` / `applyHeatMap` — results dashboard, per-finger accuracy, WPM sparkline, missed-key heatmap.
- `startSnapshots` / `startTimer` / `restartCurrentMode` — timing/snapshot loops and restart.

### 5. Event Listeners (lines 1356–1607)
- Mouse handlers for drag-and-drop key remapping.
- The big **`keydown` handler (the "Matrix Translation Engine")** — the heart of the app:
  1. Translates the physical OS key (`e.code`) → source matrix position → target-layout character. This is what lets you "test-drive" a Colemak-DH layout while your firmware is still QWERTY.
  2. Handles shortcuts: `Tab` (hold) + `Enter` = quick restart; `Escape` = end test; `Enter` after a test = new test; `Space` after a test = practice mistakes.
  3. Core typing evaluator — correctness (strict Monkeytype rules), streaks, per-finger and per-char mistake tracking.
- `keyup` — clears key-press highlights / Tab state.
- **`window.onload`** — bootstrap: version-check (wipe stale cache), `cacheDOM`, populate selects, `loadConfig`, wire up change handlers, `await loadDictionary()`, init keyboard, start first test.
- Service worker registration for PWA.

## Key concepts to understand the codebase

- **Source vs Target layout.** "Source" = what your OS/firmware currently sends. "Target" = the layout you want to learn. The keydown engine remaps source→target so you can practice a new layout without flashing firmware.
- **Position-keyed maps.** Everything keys off ZMK matrix position IDs (`LH_C4R2`, `K07`). `geo`, `pos`, and `layouts` arrays are all parallel/index-aligned per hardware.
- **No persistence backend.** All scores, heatmaps, configs, and uploaded layouts live in `localStorage`. Bumping `APP_VERSION` clears it.
- **External runtime fetches** (graceful-degrading): Monkeytype dictionary (GitHub raw), live news (NPR/NYT RSS). Offline falls back to the built-in word list.

## Working on this project

- **Edit:** open `index.html` directly. CSS in `<style>`, markup in `<body>`, logic in `<script>`.
- **Run locally:** open the file in a browser, or serve the folder (e.g. `python -m http.server`) — needed for service worker / fetch to behave like production.
- **Add a keyboard:** add an entry to `HARDWARE` with index-aligned `geo`, `pos`, and `layouts`.
- **Deploy:** push to `main`; GitHub Pages publishes automatically.
- **Branches:** `main` is the published branch. Current working branch is `yknipir`.
