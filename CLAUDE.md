# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A browser action game: hit the ball back at the right timing to keep a rally going, playing as
one of ten mascot characters (three — ふにふに / ことこと / のそのそ — have PNG pose art, the rest
are drawn as parametric SVG). The entire app — HTML, CSS, and JS — lives in
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

- **Screens**: `title`(character select) → `modes` → `ladder` → `play` → `result`, plus `howto`
  (opened from the title button) and `achievements`, toggled via `show(id)` + a `SCREENS` array.
  Any new screen must be added to `SCREENS`.
- **Modules (in dependency order)**: `Sfx` (Web Audio, ported from Rhythm_game) → `Court`/`rand`
  (logical X 0–100 mapped to CSS 15–85%, swappable RNG via `setRng`) → `scoreLabels()` (pure
  real-tennis scoring: 0/15/30/40, deuce, AD) → `Characters`/`setPose` → `ShotSystem` (timing
  tiers + course → target/duration/arc, DOM-free) → `AutoMover` → `OPPONENTS` data + `canReach`/
  `aimFromContact`/`aimAwayFromPlayer` → `Fx` (popups/flash/shake) → `TennisGame` (rAF loop, rally
  state machine, `debugState`) → `Modes` (tournament progress in localStorage) → `Input` →
  navigation.
- **Characters**: `CHARACTERS` holds 10 mascots. Each has `body`/`accent`/`ear`; ふにふに and
  ことこと additionally have `idle`/`run`/`swing` PNG paths. `poseFile()` returns a path when art
  exists and loads, otherwise `null`, and `renderCharInto()` then draws the character with
  `MochiArt.svg()` — a parametric SVG mochi (body, headband, blush, ear variant) plus an overlaid
  racket that only SVG characters need (the PNGs already have one drawn in). **To upgrade a
  character, just add its three PNG paths to its `CHARACTERS` entry** — nothing else changes.
- **`OPPONENTS`** is a data table (`char` for which mascot it wears, plus speed / reactionMs /
  reach / errorRate / smartChance / durationMul / smashReachMul / lobWeak / tellMs / smashEvery).
  All personality lives in
  `opponentArrives()`, which reads those numbers — balance changes should be table edits, not new
  branches. Each opponent has a readable "tell": a lean toward the aimed side, a red `charge` glow
  before a hard shot, and the net-rusher physically moving up the court (`depth`, weak to lobs).
- **`TennisGame` knows nothing about modes.** `begin({opponent, meta, onEnd})` reports the finished
  game through `onEnd(result)`; `Modes.Tournament` / `Modes.Rally` write progress and draw the
  result screen's buttons. Keep new modes on that seam.
- **Rally Attack** (`meta.mode === "rally"`) reuses the same rally loop with `RALLY_MACHINE`
  (reach 100, never misses) and `rampFor(hits)` for the difficulty curve; one miss ends the run.
  Scoring is `rallyPoints(tier, combo)` plus 500 every 10 hits, an `ok` scores but breaks the combo.
  The HUD swaps its two labels to SCORE / COMBO instead of the tennis points.
- **Leaderboard**: `Scores` (localStorage best) / `Leaderboard` (Firestore REST via `fetch`) /
  `RankUI` are ported from `Rhythm_game/index.html`; Rally Attack submits under gameId `tennis`,
  capped at 100000 to match the Firestore rules. A sandbox can't reach `firestore.googleapis.com`,
  so a stuck "読み込み中…" in local tests is the environment, not a bug.
- **`Progress`** holds achievements and unlocks in `tennis-game:progress`. `Progress.onResult(res)`
  runs once per finished game/run — add new achievements to the `ACHIEVEMENTS` table and a check
  there, not scattered through the game loop. Unlocks are cosmetic only (court gradient via
  `.court[data-theme]`, ball via `.ball[data-skin]`) and are applied by `applyEquipped()`.
- **Smash gauge**: perfect/good hits fill `gauge` (34 / 12), a lost point drains 20, and at 100 the
  next perfect/good automatically becomes a smash (no extra input — a down-swipe would fight the
  browser's pull-to-refresh). Lobs never consume the gauge (`keepSmash`).
- **Controls are one-thumb**: the player character auto-runs to the ball (`AutoMover`). A tap /
  Space swings at `pointerdown` time; the judgment tier (`perfect`/`good`/`ok`/`whiff`) comes from
  `tapTime - arrivalAt`. The course (left/right/straight/lob/drop) is read *after* the swing from
  the swipe direction (◀ ▶ sides, ▲ lob, ▼ drop; or a held/just-pressed arrow key) during a short
  window; a 90 ms hit-stop hides that latency. A whiff is not an instant loss — it only locks
  swinging for a cooldown, so the auto-miss timer decides the point. The one deliberate extra
  input is **net rush** (`#netBtn` / `N`): it arms "move to the net after the next hit".
- **Depth (front/back)**: every actor and every landing point carries `depth` (0 = baseline,
  1 = net). `Court.playerY/opponentY/playerBottom/opponentTop` turn it into screen position.
  Moving forward (`DEPTH_FWD_SPEED`) is faster than retreating (`DEPTH_BACK_SPEED`), which is
  what makes lob (land at depth 0) and drop (land at ~0.92) matter: both `playerCanReach()` and
  `opponentArrives()` reject a ball whose depth gap exceeds `DEPTH_REACH`. The player follows the
  incoming ball's depth automatically unless net rush is armed, in which case they stick to
  `NET_DEPTH`; the opponent lobs over a player at the net (`lobOverPlayer`). The old `atNet` /
  `setNet` are now thin wrappers over depth. Opponent lateral speed is scaled to *arrive with the
  ball* (`need = dist / remain`) rather than sprinting and freezing — keep that when touching
  `updateOpponent()`.
- **Audio**: `Sfx` owns one `AudioContext`. Effects are one-shot `tone()`s; the BGM is a 6-layer
  look-ahead scheduler inside the same module (`startBgm/stopBgm/bgmStats`; ~42 nodes/s measured,
  schedule 0.6 s ahead, abandon catch-up when behind — same lessons as neon-void). `setMuted()`
  drives `master.gain` and is persisted in `tennis-game:muted`; the 🔊 button is global.
- **Rally phases**: `serve → ballToPlayer → hitstop → ballToOpponent → returning → ballToPlayer …`,
  ending in `pointOver` / `gameOver`; `ballOut` is the opponent's shot sailing wide. Ball position
  is lerped each frame with a fake arc (`hover = arc*4t(1-t)`); logic uses the lerped x, the arc is
  render-only.
- **Writing Playwright tests**: after the opponent returns, the incoming ball is *already* in
  `ballToPlayer`. Waiting for a ball with a newer `flightStart` skips it and the auto-miss timer
  takes the point — a bug that has bitten this repo repeatedly. Swing at the state you already
  have.
- **`window.__tennis`**: `begin/swing/applyCourse/forceCourse/startMode/tournament/opponents/
  setRng/scoreLabels/debugState/sfx`, plus `scores`, `rallyPoints`, `rampFor`. Tests normally start
  a match with `startMode("tournament", {stage: n, force: true})` so they can pick any opponent
  directly, or `startMode("rally")`.
  `debugState()` exposes phase, ball (incl. `arrivalAt`), player/opponent, `lastShot`
  (tier/deltaMs/course), score labels and the timing constants, so Playwright can compute exactly
  when to dispatch a `pointerdown` on `#court` — same idea as `debugState()` on `EchoGame`/`RingGame`
  in the `Rhythm_game` repo. Tests dispatch `pointermove` with `clientX/clientY` deltas to swipe.
- **Character art**: `art/<char>-{idle,run,swing}.png` are 1024×1024 RGBA sprites generated
  with fal.ai (gpt-image-2) in masa's local session. gpt-image-2 returns *opaque* PNGs even when
  asked for transparency, so the background was stripped afterwards with a border-connected
  flood fill — do the same for any new character. `art/funifuni.png` / `kotokoto.png` (60×60 crops
  from LINE sticker sheets) remain only as the `poseFile()` fallback. `art/title-bg.png` is the
  app-wide background (`#app`), dimmed under `#play`.
- **Pause**: every `setTimeout` inside `TennisGame` that matters for correctness goes through
  `pt(name, fn, delay)`, which records `{fn, fireAt}` in `timerMeta[name]` alongside the native
  timer id. `pause()` cancels each named timer and freezes `paused=true`; `resume()` shifts every
  absolute timestamp (`ball.flightStart`, `opponentReactAt`, `*cooldownUntil`, `*SwingUntil`,
  `hitstopUntil`, `courseWindowUntil`) forward by the paused duration and reschedules each timer
  for its *remaining* delay via the same `pt()`. `swing()`/`applyCourse()` both bail out early when
  `paused`. The outer glue calls `TennisGame.pause()` on `visibilitychange` (hidden) and never
  auto-resumes — the player must press "再開する". Adding a new mid-rally timer means routing it
  through `pt()` too, or it will silently keep running while "paused".
- **Best-of-N matches**: `TennisGame` still only knows about one game (`begin()` → `onEnd(result)`).
  `Modes.Tournament` wraps that in a match: `matchScore` (module-local) counts games won per side,
  `MATCH_WINS = 2`. `onEnd` increments it and, while undecided, shows a "次のゲームへ" button
  (`Tournament.nextGame()`) instead of progressing — tournament advancement, `beaten` counts, and
  the achievement-relevant `matchWin` flag only apply once someone reaches `MATCH_WINS`. Per-game
  achievements (`love_game`, `no_miss_game`, etc.) still fire every game via `Progress.onResult()`,
  unchanged. Rally Attack is untouched (still a single endless run).

## Scope

The rebuild is complete: one-thumb controls with shot tiers, a five-opponent tournament ladder with
readable tells, a smash gauge, an endless Rally Attack with the shared Firestore TOP10, and
achievements/unlocks with hit particles, screen shake and judgment popups. Character art is the
sporty `art/<char>-{idle,run,swing}.png` set (fal.ai, generated in masa's local session, downscaled
to 320px here); `art/funifuni.png` / `kotokoto.png` remain as the `poseFile()` fallback.

Since then: BGM, a global mute, depth/forward movement with lob/drop/net-rush, a howto screen,
PNG art for のそのそ, per-character stats, opponent drop shots, a perspective zoom on depth, a
pause (button / Escape / tab-hide) that correctly rewinds every in-flight timer, and best-of-2
matches in Tournament mode. Still out: seven characters remain SVG-only (no PNG art), and
opponents don't have per-character stat panels the way the player does.
- **Character stats** live on each `CHARACTERS` entry as `stats:{reach,timing,gauge,foot}` (1–5,
  3 = neutral, every character totals 12). `playerStatsFor()` turns them into multipliers that
  only touch the *player*: reach radius, `ShotSystem.judge(delta, winMul)` window width, gauge gain,
  and `stepDepth()` speed. Opponents are unaffected — their feel comes from `OPPONENTS`. The
  selected character's stats are shown under the title grid (`#charDetail`).
- **Opponent drop shots**: `OPPONENTS[].dropChance` — when the player is deep (`depth < 0.3`)
  the opponent may drop to the net (low arc 30 is the tell). `RALLY_MACHINE` has `dropChance:0`
  so leaderboard difficulty is unchanged.
