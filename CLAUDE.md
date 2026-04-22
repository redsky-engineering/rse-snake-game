# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install       # install dependencies
npm run dev       # start dev server (Vite HMR)
npm run build     # production build → dist/
npm run preview   # serve the dist/ build locally
```

No test runner or linter is configured.

## Architecture

Single-component Svelte 4 app, built with Vite. All game logic lives in one file:

- `src/main.js` — mounts `App.svelte` into `#app`
- `src/App.svelte` — layout shell: header with logo + `<SnakeGame />`
- `src/lib/SnakeGame.svelte` — the entire game: logic, rendering loop, and styles
- `src/app.css` — global reset and body background

### Rendering model

`SnakeGame.svelte` uses a **fixed-timestep + RAF interpolation** approach:

- `TICK_MS = 150` drives game logic at a fixed rate via an accumulator
- `requestAnimationFrame` runs every frame; `tickLogic()` is called as many times as the accumulator allows
- `getDrawSegments(t)` lerps each snake segment between its previous and current grid positions using `t = accumulator / TICK_MS`, giving sub-tick smooth motion

### Canvas layers

- **`bgCanvas`** — offscreen canvas pre-rendered once at mount (and on resize): hex-tile grid + faint logo watermark. Blitted each frame with a single `drawImage` call.
- **main `canvas`** — game objects drawn on top each frame: food (pulsing glow), snake body (layered strokes with gradient), snake head (corona + shine + eyes).

### State

All state is plain `let` variables scoped to the component (no Svelte stores). Key variables:

| Variable | Purpose |
|---|---|
| `snake` | Array of `{x,y}` grid cells, head-first |
| `direction` / `nextDirection` | Separation prevents 180° reversal mid-tick |
| `prevSnake` | Snapshot before each tick, used for interpolation |
| `accumulator` / `lastTimestamp` | Fixed-timestep bookkeeping |

### Marked extension points

The codebase has `// FUTURE:` comments at the natural spots for common feature additions:

- `spawnFood` → power-up types
- `tickLogic` → score multipliers / level-up
- `endGame` → high score persistence
- `TICK_MS` constant → difficulty levels

### Sizing

`computeCellSize()` derives `CELL_SIZE` from viewport dimensions on mount and after a debounced resize. `CANVAS_W` / `CANVAS_H` are reactive (`$:`) so Svelte automatically updates the `<canvas>` attributes, after which `buildBackground()` must be called manually to re-render the offscreen canvas at the new size.

## Brand / Visual conventions

Colors are RedSky brand: `#cc1414` (snake body), `#e84040` (head/hover), `#2563eb` (food), `#0c0c10` (background). The logo at `public/logo.svg` is used as a watermark on the game canvas.
