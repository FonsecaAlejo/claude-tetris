# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A single-file Tetris implementation in vanilla JavaScript (ES6+), HTML5 Canvas, and CSS. No dependencies, no build process, no package.json.

## Running the game

There is no build/lint/test tooling. To run:

```bash
open index.html                # macOS, opens directly in browser
# or serve statically:
python3 -m http.server 8000    # then visit http://localhost:8000
npx serve .
```

Since there's no test suite, verify changes by opening the game in a browser and playing it (movement, rotation, line clears, level/speed changes, pause, game over/restart).

## Architecture

Three files, all logic lives in `game.js` (~300 lines, no modules/classes — flat functions and top-level mutable state):

- `index.html` — DOM structure: `#board` canvas (300×600, the 10×20 grid at `BLOCK=30`px/cell), `#next-canvas` (piece preview), HUD spans (`#score`, `#lines`, `#level`), and the `#overlay` used for both PAUSE and GAME OVER states.
- `style.css` — dark/retro arcade visual theme.
- `game.js` — entire game logic and state.

### Key model in game.js

- **Board**: `board` is a `ROWS × COLS` matrix; each cell is `0` (empty) or a piece color index `1–7`.
- **Pieces**: `PIECES` are square matrices of color indices; `current` and `next` are `{ type, shape, x, y }`. Rotation is `rotateCW` (transpose + reverse), applied via `tryRotate`, which attempts wall kicks at offsets `[0, -1, 1, -2, 2]` before giving up.
- **Collision**: `collide(shape, ox, oy)` checks board bounds and cell overlap — the single gate used for movement, rotation, spawn, and ghost projection.
- **Game loop**: `loop(ts)` runs via `requestAnimationFrame`, accumulates `dropAccum` and advances the piece one row (or calls `lockPiece()`) once `dropAccum >= dropInterval`.
- **Locking**: `lockPiece()` → `merge()` (writes piece into `board`) → `clearLines()` → `spawn()` (promotes `next` to `current`, generates a new `next`; if the new piece immediately collides, calls `endGame()`).
- **Line clearing/scoring**: `clearLines()` scans bottom-up, splices full rows and unshifts empty ones at the top. Score uses `LINE_SCORES = [0, 100, 300, 500, 800]` × `level`; hard drop adds 2 pts/row dropped, soft drop adds 1 pt/row. `level = floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level - 1) * 90)`.
- **Ghost piece**: `ghostY()` projects `current` straight down via repeated `collide` checks; drawn with `globalAlpha = 0.2`.
- **Rendering**: `draw()` clears and redraws the grid, locked board, ghost piece, and current piece every frame; `drawNext()` renders the preview canvas.
- **Input**: a single `keydown` listener handles arrows (move/soft-drop), `ArrowUp`/`KeyX` (rotate), `Space` (hard drop, `preventDefault`ed), `KeyP` (pause toggle via `togglePause()`, which also drives the overlay).
- **Init/restart**: `init()` resets all state (`board`, `score`, `lines`, `level`, `dropInterval`, etc.) and restarts the RAF loop; bound to the `#restart-btn` click handler and called once at file load.

### Tunable constants (top of game.js)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS × BLOCK` and `ROWS × BLOCK`).
