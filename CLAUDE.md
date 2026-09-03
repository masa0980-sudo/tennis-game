# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A browser action game: hit the ball back at the right timing to keep a rally going, playing as
one of two mascot characters (ふにふに / ことこと). The entire app — HTML, CSS, and JS — lives in
a single `index.html` file with no build step, no bundler, and no dependencies, following the
same philosophy as the sibling `Rhythm_game` repo. Published as-is to GitHub Pages.

## Commands

There is no package.json, build step, linter, or test framework — this is intentional; keep it
that way rather than introducing tooling.

- **Run locally**: serve the repo root and open `index.html`, e.g. `python3 -m http.server 8000`.
- **Verify UI changes**: use Playwright to drive a real browser against the local server (there is
  no automated test suite). The browser is pre-installed — launch with
  `executablePath: "/opt/pw-browsers/chromium"` and do **not** run `playwright install`.
- **Deploy**: push to `main`. GitHub Actions (`.github/workflows/deploy-pages.yml`) rebuilds and
  republishes GitHub Pages automatically — there is no separate deploy command.

## Architecture

Single `<script>` inside `index.html`, following the same numbered-module style as
`Rhythm_game/index.html` (small helpers, then the game's own IIFE module, then screen-switching
glue at the bottom).

- **Screens**: `title`(character select) → `play` → `result`, toggled via `show(id)` +
  a `SCREENS` array. Any new screen must be added to `SCREENS`.
- **Modules (in dependency order)**: `Sfx` (Web Audio, ported from Rhythm_game) → `Court`/`rand`
  (logical X 0–100 mapped to CSS 15–85%, swappable RNG via `setRng`) → `scoreLabels()` (pure
  real-tennis scoring: 0/15/30/40, deuce, AD) → `Characters`/`setPose` → `ShotSystem` (timing
  tiers + course → target/duration/arc, DOM-free) → `AutoMover` → opponent data + `canReach`/
  `aimFromContact`/`aimAwayFromPlayer` → `Fx` (popups/flash) → `TennisGame` (rAF loop, rally
  state machine, `debugState`) → `Input` → navigation.
- **Controls are one-thumb**: the player character auto-runs to the ball (`AutoMover`). A tap /
  Space swings at `pointerdown` time; the judgment tier (`perfect`/`good`/`ok`/`whiff`) comes from
  `tapTime - arrivalAt`. The course (left/right/straight/lob) is read *after* the swing from the
  swipe direction (or a held/just-pressed arrow key) during a short window; a 90 ms hit-stop hides
  that latency. A whiff is not an instant loss — it only locks swinging for a cooldown, so the
  auto-miss timer decides the point.
- **Rally phases**: `serve → ballToPlayer → hitstop → ballToOpponent → returning → ballToPlayer …`,
  ending in `pointOver` / `gameOver`. Ball position is lerped each frame with a fake arc
  (`hover = arc*4t(1-t)`); logic uses the lerped x, the arc is render-only.
- **`window.__tennis`**: `begin/swing/applyCourse/forceCourse/setRng/scoreLabels/debugState/sfx`.
  `debugState()` exposes phase, ball (incl. `arrivalAt`), player/opponent, `lastShot`
  (tier/deltaMs/course), score labels and the timing constants, so Playwright can compute exactly
  when to dispatch a `pointerdown` on `#court` — same idea as `debugState()` on `EchoGame`/`RingGame`
  in the `Rhythm_game` repo. Tests dispatch `pointermove` with `clientX/clientY` deltas to swipe.
- **Character art** (`art/funifuni.png`, `art/kotokoto.png`): single poses cropped out of existing
  LINE sticker sheets (`mo-portfolio/img/stamp_01_funifuni2026.png` /
  `stamp_02_kotokoto2026.png`), background kept transparent. The source stamps are only 60×60px
  per pose, so don't render the character much larger than ~96–120px on screen or it'll look soft.
  Higher-resolution originals exist only on masa's local machine — ask before assuming they're
  available anywhere else.

## Scope / roadmap

The game is being rebuilt in phases (see the commit history): 1) one-thumb controls + shot tiers,
2) a ladder of opponents with distinct personalities, a smash gauge and a tournament mode,
3) an endless "rally attack" score mode with the shared Firestore TOP10 (same mechanism as
Rhythm_game, gameId `tennis`), 4) achievements/unlocks and presentation (particles, shake,
slow-mo, court themes). Sporty 4-pose character art (`art/<char>-<pose>.png`) has been requested
from masa's local image-generation session; until it lands, poses are CSS-only (`data-pose`).
