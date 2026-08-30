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
- **`TennisGame`**: the core timing loop. A ball animates toward the player over `duration` ms;
  tapping/clicking/pressing Space within `[duration - WINDOW_MS, duration + GRACE_MS]` of launch
  counts as a hit, extends the rally, and speeds things up for the next volley. Missing (wrong
  timing, or no input at all) ends the run.
- **`window.__tennis`**: exposes `debugState()` (phase/duration/launchAt/windowMs/graceMs/rally)
  for Playwright to compute exactly when to dispatch a tap — same idea as `debugState()` on
  `EchoGame`/`RingGame` in the `Rhythm_game` repo.
- **Character art** (`art/funifuni.png`, `art/kotokoto.png`): single poses cropped out of existing
  LINE sticker sheets (`mo-portfolio/img/stamp_01_funifuni2026.png` /
  `stamp_02_kotokoto2026.png`), background kept transparent. The source stamps are only 60×60px
  per pose, so don't render the character much larger than ~96–120px on screen or it'll look soft.
  Higher-resolution originals exist only on masa's local machine — ask before assuming they're
  available anywhere else.

## Scope

This is a first pass: only the core timing mechanic and character selection are in. Rally/hit
effects, BGM, an animated opponent, combo scoring, etc. are intentionally left for later — don't
add them unprompted.
