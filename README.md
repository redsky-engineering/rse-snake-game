# Snake Game — AI-Assisted Feature Development

This is a starter Snake game built with Svelte + Vite. The core gameplay is intentionally minimal. The goal of this project is to practice using AI tools (like Claude) to plan, implement, and iterate on new features.

Each feature below is a self-contained addition to the base game — a good exercise in prompting AI to extend existing code without breaking what's already there.

## Getting Started

```bash
npm install
npm run dev
```

## Suggested Features

Work through these individually or in groups. For each one, try to write a clear prompt describing what you want, then let AI help you implement it.

### Core Enhancements
- **High score persistence** — save the all-time best score to `localStorage` and display it alongside the current score
- **Pause/resume** — press Space to pause and unpause the game mid-play
- **Game speed control** — let the player pick Easy / Normal / Hard before starting, each with a different tick rate
- **Progressive speed increase** — automatically speed up the snake as the score climbs
- **Game over screen with stats** — show final score, snake length, and time survived on the game over overlay
- **Grid size options** — let the player choose between small (15×15), medium (20×20), and large (30×30) grids

### Visual & Audio
- **Sound effects** — play a sound when food is eaten, and a different one on game over
- **Arrow visualization / direction preview** — draw a small indicator showing which direction the snake is currently heading

### Multiplayer / Social
- **Leaderboard** — store the top 10 scores locally (or via a simple backend) and show a leaderboard screen

### Game Modes
- **Wrap mode** — instead of dying at the wall, the snake wraps around to the opposite side
- **Obstacles mode** — place static walls on the grid that the snake must navigate around

### Power-up Pills
Spawn collectible pills on the grid alongside the food. Each has a timed effect:

| Power-up | Effect |
|---|---|
| Speed boost | Temporarily increases snake speed |
| Slow motion | Temporarily decreases snake speed |
| Shrink | Reduces the snake's current length |
| Grow | Increases the snake's current length |
| Double points | 2× score multiplier for a short duration |
| Invincibility | Temporarily ignore wall and self-collision |
| Freeze | Pauses all game movement briefly |
