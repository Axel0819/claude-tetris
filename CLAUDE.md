# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Classic Tetris implemented in vanilla JavaScript with HTML5 Canvas and CSS. No dependencies, no build step, no package.json — just three files: `index.html`, `style.css`, `game.js`.

## Running the game

There is no build/install/test tooling. Just open or serve `index.html`:

```bash
start index.html        # Windows, opens directly in default browser
python3 -m http.server 8000   # or serve locally, then visit http://localhost:8000
npx serve .
```

Verify changes by opening the page in a browser and playing — there is no automated test suite.

## Architecture

All game logic lives in `game.js` (~300 lines) as top-level module-scope state and functions (no classes, no build/module system — plain script tag, global scope).

- **Board model**: `board` is a `ROWS × COLS` (20×10) matrix; each cell is `0` (empty) or a piece color index `1–7`.
- **Pieces**: `PIECES` defines the 7 tetrominoes as square matrices (indices matching `COLORS`). Rotation is done by `rotateCW` (transpose + reverse rows), not by predefined rotation states.
- **Collision** (`collide`): checks board bounds and existing fixed blocks for a given shape/offset.
- **Wall kicks** (`tryRotate`): after rotating, tries offsets `[0, -1, 1, -2, 2]` columns until a non-colliding position is found.
- **Game loop** (`loop`): driven by `requestAnimationFrame`; accumulates elapsed time in `dropAccum` and advances the piece one row once `dropInterval` is exceeded.
- **Locking/clearing**: `lockPiece` → `merge` (bakes current piece into `board`) → `clearLines` (removes full rows bottom-up, unshifts empty rows at top, updates score/level/dropInterval).
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` multiplied by `level`; hard drop adds 2 pts/row dropped, soft drop adds 1 pt/row.
- **Leveling/speed**: level increases every 10 lines; `dropInterval = max(100, 1000 - (level-1)*90)` ms.
- **Ghost piece**: `ghostY` projects the current piece straight down to its landing row; drawn at `globalAlpha = 0.2` in `draw()`.
- **Rendering**: `draw()` redraws the whole board + grid + ghost + current piece every frame onto `#board` canvas; `drawNext()` renders the preview piece onto the separate `#next-canvas`.
- **Input**: single `keydown` listener switches on `e.code` (arrows, `KeyX` for rotate, `Space` for hard drop, `KeyP` for pause). Movement/rotation are ignored while `paused` or `gameOver`.
- **Game over / restart**: `spawn()` calls `endGame()` if the newly spawned piece immediately collides; the overlay's restart button calls `init()` to fully reset state.

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK` (cell size px), `COLORS`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS×BLOCK` by `ROWS×BLOCK`).
