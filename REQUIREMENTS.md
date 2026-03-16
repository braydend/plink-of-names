# Plink of Names — Requirements & Plan

## Overview

A web application that randomly selects a name from a user-provided list using a visually engaging "Plinko"-style animation. A ball drops from the top of the screen, bounces off pegs, and lands in a bucket labeled with the selected name.

Built with: **SvelteKit 2 / Svelte 5**, **Tailwind CSS 4**, **TypeScript**, **Matter.js** (physics engine)

---

## Features

### 1. Name List Input

- Two ways to enter names:
  - **Bulk entry**: A textarea where users can paste/type multiple names (one per line)
  - **Individual entry**: A single input field with an "Add" button to add names one at a time
- Names can be removed individually from the list
- Minimum of 2 names required to drop a ball
- Names stay in the pool after being selected (no elimination mode)
- No persistence across sessions (fresh list each visit)

### 2. Plinko Board

- A visual board with rows of pegs arranged in a classic Plinko triangle pattern
- A ball drops from a central point at the top
- The ball bounces off pegs with realistic physics powered by **Matter.js**
- At the bottom, "buckets" are labeled with names from the user's list
- **Adaptive sizing**: The board fills available screen space, scaling peg rows and spacing to fit
- The result is **physics-determined** — wherever the ball lands is the winner (no pre-selection)
- **Randomised bucket positions**: Name-to-bucket assignments are shuffled on each drop to avoid positional bias

### 3. Ball Drop Interaction

- User clicks a "Drop" button to release the ball
- The ball's path is fully determined by physics simulation — genuine randomness from each peg collision
- The final bucket the ball lands in determines the selected name

### 4. Result Display

- When the ball lands, the camera **zooms in** on the winning bucket and ball
- The winning bucket is **highlighted**
- A **celebration animation** plays (e.g. confetti, glow, particle effects)
- The winning name is displayed prominently

### 5. Visual Design

- **Colourful and playful** theme — bold colours, fun feel
- Desktop-only layout for now
- Canvas-based rendering for the Plinko board

---

## Implementation Plan

### Phase 1 — Foundation ✅

- [x] Install Matter.js dependency
- [x] Set up page layout with two panels: name input (sidebar) and Plinko board (main area)
- [x] Build name list input component:
  - [x] Textarea for bulk name entry (one per line)
  - [x] Single input + "Add" button for individual names
  - [x] List display with remove buttons for each name
- [x] Store name list in Svelte 5 reactive state (`$state`)

### Phase 2 — Plinko Board Rendering ✅

- [x] Create a canvas-based Plinko board component
- [x] Set up Matter.js world with gravity
- [x] Calculate and render peg positions in a triangular grid pattern
- [x] Render labelled buckets at the bottom, one per name
- [x] Adaptive board sizing — adjust peg rows and board dimensions based on name count
- [x] Add walls/boundaries to contain the ball
- [x] Colourful visual styling for pegs, ball, and buckets

### Phase 3 — Ball Physics & Animation ✅

- [x] Implement "Drop" button (disabled when < 2 names or ball is in flight)
- [x] Create ball body in Matter.js and drop from top-centre
- [x] Ball collides with pegs and bounces realistically
- [x] Detect when ball settles into a bucket
- [x] Map bucket position to the corresponding name

### Phase 4 — Result & Polish ✅

- [x] Animate a smooth zoom-in on the winning bucket area when ball lands
- [x] Highlight the winning bucket on selection
- [x] Display winning name prominently (overlay or announcement area)
- [x] Celebration animation (confetti / particles)
- [x] "Drop Again" button to reset and go again
- [x] Smooth transitions and visual feedback throughout

### Stretch Goals (Future)

- [ ] localStorage persistence for name lists
- [ ] Sound effects (peg hits, landing, celebration)
- [ ] Mobile-responsive layout
- [ ] Selection history panel
- [ ] Theming / colour customization
