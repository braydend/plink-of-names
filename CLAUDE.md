# Plink Of Names

A Plinko-style name selector: drop a ball through pegs into named buckets to randomly pick a name.

## Tech Stack

- **SvelteKit 2** with **Svelte 5** (runes: `$state`, `$derived`, `$effect`, `$props`, `$bindable`)
- **Tailwind CSS 4** (via Vite plugin, no PostCSS config)
- **TypeScript**
- **Matter.js** for physics (ball/peg collisions, gravity, settle detection)
- **HTML Canvas** for rendering (custom render loop, not Matter.js renderer)

## Commands

- `npm run dev` — start dev server
- `npm run build` — production build
- `npm run check` — svelte-check type checking
- `npm run lint` — prettier + eslint
- `npm run format` — auto-format with prettier
- `npm run test` — run vitest tests

## Architecture

### Key Components

- `src/routes/+page.svelte` — Main page. Manages all app state (names, settings, dropping state), localStorage persistence, and two-panel layout (sidebar + board).
- `src/lib/components/PlinkoBoard.svelte` — Core canvas component. Owns Matter.js engine, camera system, rendering, and physics. Exports `dropBall()`, `resetBall()`, `randomise()`.
- `src/lib/components/NameListInput.svelte` — Name input UI with single add, bulk paste, and remove. Uses `$bindable` for two-way `names` binding.

### World vs View Coordinate System

PlinkoBoard separates physics world dimensions from canvas viewport dimensions. The world can be taller than the viewport (via `boardScale`). The canvas always matches the container size; a camera system (translate + scale) pans over the world. Three camera modes:
1. **Scale-to-fit** (idle, world > viewport) — scales world down to show everything
2. **Ball-follow** (during drop, world > viewport) — zoomed 1.8x, tracks ball with adaptive lerp
3. **Result zoom** (after settle) — 2.5x zoom into winning bucket with confetti

### Physics Notes

- Solver iterations set to 10 (position + velocity) to prevent ball tunneling
- Floor is 40px thick to catch fast balls
- Walls positioned at `SIDE_PADDING - PEG_RADIUS - BALL_RADIUS` to prevent ball wedging between wall and edge pegs
- Settle detection: speed < 0.3 for 30 frames, or 3s fallback in bucket area
- Bucket names shuffled via Fisher-Yates on each `buildBoard()` call

## Conventions

- Svelte 5 runes only — no `$:` reactive statements, no stores
- British English in UI copy and code comments (colour, randomise, etc.)
- Components use `$state` for canvas/container element refs
- Guard `$effect` and `ResizeObserver` callbacks with `!dropping && !isZooming` to prevent mid-animation rebuilds
