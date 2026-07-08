# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Classic Tetris implemented in vanilla JavaScript with HTML5 Canvas and CSS. No dependencies, no framework, no build step. The three source files (`index.html`, `style.css`, `game.js`) load directly in the browser.

## Running

No install or compile step. Either open `index.html` directly, or serve statically to avoid file:// quirks:

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

There is no test suite, linter, or build tooling — changes are verified by playing in the browser.

## Architecture (`game.js`)

Everything lives in one file built around module-level mutable state (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, drop timing). Key conventions to preserve when editing:

- **Piece encoding**: `PIECES[1..7]` are shape matrices whose non-zero cells hold the _color index_ (1–7) into `COLORS`, so a cell value simultaneously means "filled" and "which color". `board` uses the same encoding (0 = empty). `drawBlock` treats `colorIndex` 0/falsy as empty and skips it — rely on this rather than adding separate occupancy checks.
- **Coordinates**: `current` holds `{ type, shape, x, y }`; `x`/`y` are the top-left board cell of the shape matrix. `collide(shape, ox, oy)` is the single source of truth for wall/floor/stack overlap and is reused for movement, rotation, ghost projection, spawn (game-over test), and gravity — call it rather than reimplementing bounds logic.
- **Rotation**: `rotateCW` returns a new matrix; `tryRotate` applies basic wall kicks by testing horizontal offsets `[0,-1,1,-2,2]`. This is not full SRS.
- **Game loop**: `loop(ts)` is a `requestAnimationFrame` loop driven by `dropAccum`/`dropInterval` (accumulated ms vs. gravity interval). Pause/game-over stop the loop via `cancelAnimationFrame(animId)`; `togglePause` restarts it and must reset `lastTime` to avoid a large `dt` jump. `dropInterval` shrinks with level in `clearLines`.
- **Piece lifecycle**: `lockPiece` → `merge` (stamp shape into board) → `clearLines` → `spawn` (promotes `next` to `current`, generates a new `next`, and calls `endGame` if the fresh piece already collides).
- **Rendering**: `draw` clears and redraws grid, settled board, ghost piece (`ghostY` + 0.2 alpha), then the active piece each frame. HUD (`score`/`lines`/`level`) updates via `updateHUD` on state changes, separate from canvas rendering.

`init()` resets all state and (re)starts the loop; it is both the initial entry point and the restart-button handler. The DOM element lookups at the top of the file must stay in sync with the `id`s in `index.html`.

## Cross-file constraints

- **Canvas size is coupled to the grid constants**: the board `<canvas>` in `index.html` is hard-sized `width="300" height="600"`, which must equal `COLS * BLOCK` × `ROWS * BLOCK` (10×30 by 20×30). Change any of `COLS`/`ROWS`/`BLOCK` in `game.js` and you must update the canvas attributes to match, or rendering clips/overflows. (The `next-canvas` is `120×120` = a 4×4 preview at `BLOCK`/`NB` = 30.)
- **User-facing text is Spanish** (`Puntuación`, `PAUSA`, `Reiniciar`, control labels in `index.html`) while code identifiers are English — keep new UI strings in Spanish for consistency.
