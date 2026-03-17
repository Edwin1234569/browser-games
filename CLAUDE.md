# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A collection of self-contained browser games. Each game is a **single HTML file** with no build step, no dependencies, and no package manager — everything (HTML, CSS, JS) lives inline. Open any file directly in a browser to play.

## Running / Testing

No build or install step. Open files directly:

```bash
# Windows — open in default browser
start shooter.html
start tictactoe.html
```

There are no automated tests. Validation is done by playing the game in-browser and checking the browser console for errors.

## Git Workflow

Remote: `https://github.com/Edwin1234569/browser-games` (branch: `main`)

**After every meaningful change, commit and push immediately** — do not batch unrelated changes into one commit. This keeps the history clean and every state recoverable.

```bash
git add <file>
git commit -m "short imperative summary

Optional body explaining why, not what.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
git push
```

**Commit message rules:**
- First line: imperative mood, ≤ 72 chars (e.g. `Add shield power-up to shooter`, not `Added` or `Adding`)
- Use the body for *why* something changed if it isn't obvious from the diff
- Always include the `Co-Authored-By` trailer above

**When to commit** — treat each of these as its own commit:
- A new feature or mechanic is working
- A bug is fixed
- A visual/audio change is complete
- A new game file is added
- CLAUDE.md or repo config is updated

Never leave the working tree dirty at the end of a session.

## Architecture

### Shared conventions across both games

- **Single-file**: all CSS in `<head>`, all JS in one `<script>` block at the bottom of `<body>`.
- **Google Fonts CDN** (`Press Start 2P`) for the retro pixel font — no local font files.
- **No external JS libraries**. Canvas 2D API only.
- All game logic runs on `requestAnimationFrame` via a **fixed-timestep accumulator** loop:
  ```js
  accum += rawDelta;
  while (accum >= FIXED_STEP) { update(FIXED_STEP); accum -= FIXED_STEP; }
  render();
  ```

---

### `shooter.html` — SURVIVOR (top-down shooter)

**Coordinate system:** Canvas is `960×540` CSS pixels. All game logic runs at **logical scale** (`LW=480, LH=270`). A `ctx.scale(2, 2)` call at the start of each game-world draw pass upscales everything 2×. Mouse coordinates must be divided by `SC=2` after correcting for canvas bounding rect. All entity positions, radii, and speeds are in logical units.

**State machine** (`G.state`): `title → playing → levelup → defeated | win`. The global `G` object is replaced wholesale via `newGame()` on restart — never patch it partially.

**18 numbered sections** (follow this order when adding code):
1. Constants — tweak gameplay numbers here (`PLAYER_SPD`, `BULLET_SPD`, `LEVELS` array, `ET` enemy type configs)
2. Canvas + Input — keyboard (`KEY`/`JUST`) and mouse state; `JUST` is cleared at the end of every update tick via `clearJust()`
3. Utilities
4. Audio — all Web Audio synthesis (`playDefeatedSound`, `playImpactThud`, `playShootSound`); AudioContext is lazy-created on first use via `getAC()`
5. Particles — simple pool in `PARTS[]`; `spawnParticles(x,y,col,count,spd1,spd2,life1,life2,sz1,sz2)`
6. Bullets — object pool `BPOOL[120]`; recycles by `active` flag; `resetBullets()` on level start
7. Enemy class — `state`: `alive|hurt|dying|dead`; drawing is entirely canvas primitives per type (`grunt/fast/tank`)
8. Player class — `anim` state: `idle|walk|shoot`; gun arm rotated around a shoulder pivot using `ctx.save/translate/rotate/restore`
9. Game state — `G = newGame()` factory; `lvlCfg()` returns current `LEVELS[G.levelIdx]`
10. Spawner — respects `kills >= cfg.kill` to stop spawning; uses a `spawnQ` batch queue for staggered multi-spawns
11. Collision — circle-circle via `dist2`; bullets→enemies then enemies→player
12–14. Background / HUD / Scanlines — pure draw helpers, no state mutation
15. Screens — one `update*` + `draw*` pair per state; defeated slow-mo applies `dt * scale` to enemies/bullets/particles only, not to the defeated timer itself
16–18. Update dispatch / render dispatch / game loop / init

**Adding a new enemy type:** add an entry to `ET`, update the `drawEnemy` section inside `Enemy.draw()`, and add the type name to the relevant `LEVELS` entries.

**Adding a new level:** append to the `LEVELS` array. The game ends (win screen) when `G.levelIdx >= LEVELS.length` after a level completes.

---

### `tictactoe.html` — Tic Tac Toe

**No canvas — pure DOM.** The board is 9 `.cell` divs in a CSS Grid. All visual effects (confetti, floating symbols, win banner) are CSS animations triggered by class toggling.

**Key functions:**
- `cpuMove()` — dispatches to random (easy), threat-detection (medium), or `minimax()` (hard)
- `minimax(b, player, depth)` — full negamax on the 9-cell board array; returns `{ score, index }`
- `resolveTurn()` — checks `checkBoard()`, updates scores, triggers win celebration or hands off to `doCpuTurn()`
- `doCpuTurn()` — delays the CPU move with `setTimeout` (350–500 ms depending on difficulty) and locks the board during that window

**Difficulty** is set via `.diff-btn` buttons. Changing difficulty calls `init()` immediately, resetting the board but preserving cumulative scores.

**Win celebration sequence:** `doFlash()` → `showWinBanner()` → `spawnConfetti(180)` + `tickConfetti()` + `burstFloaties()`. Confetti runs on its own `requestAnimationFrame` loop (`confettiAnim` handle) separate from any game loop; `stopConfetti()` cancels it.
