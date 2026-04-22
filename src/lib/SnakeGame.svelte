<script>
  import { onMount, onDestroy } from 'svelte'

  // --- Grid & timing ---
  // FUTURE: expose TICK_MS as a prop to implement difficulty levels
  const GRID_COLS = 20
  const GRID_ROWS = 20
  const CELL_SIZE = 20
  const CANVAS_W = GRID_COLS * CELL_SIZE
  const CANVAS_H = GRID_ROWS * CELL_SIZE
  const TICK_MS = 150

  // --- Colors ---
  const COLOR_BG    = '#0f3460'
  const COLOR_GRID  = '#1a4a7a'
  const COLOR_FOOD  = '#e94560'
  const COLOR_SNAKE = '#4ade80'
  const COLOR_HEAD  = '#86efac'

  // --- Game state ---
  let snake = []
  let direction = { x: 1, y: 0 }
  let nextDirection = { x: 1, y: 0 }
  let food = { x: 0, y: 0 }
  let score = 0
  let gameOver = false
  let started = false

  // --- Canvas ---
  let canvas
  let ctx
  let intervalId = null

  function initGame() {
    const midY = Math.floor(GRID_ROWS / 2)
    snake = [
      { x: 12, y: midY },
      { x: 11, y: midY },
      { x: 10, y: midY }
    ]
    direction = { x: 1, y: 0 }
    nextDirection = { x: 1, y: 0 }
    score = 0
    gameOver = false
    started = false
    spawnFood()
  }

  function spawnFood() {
    // FUTURE: return { x, y, type } here to support power-ups
    let pos
    do {
      pos = {
        x: Math.floor(Math.random() * GRID_COLS),
        y: Math.floor(Math.random() * GRID_ROWS)
      }
    } while (snake.some(seg => seg.x === pos.x && seg.y === pos.y))
    food = pos
  }

  function handleKeyDown(event) {
    const keyMap = {
      ArrowUp:    { x: 0,  y: -1 },
      ArrowDown:  { x: 0,  y:  1 },
      ArrowLeft:  { x: -1, y:  0 },
      ArrowRight: { x: 1,  y:  0 }
    }

    const mapped = keyMap[event.key]
    if (!mapped) return

    event.preventDefault()

    // Ignore reverse — would cause immediate self-collision
    if (mapped.x === -direction.x && mapped.y === -direction.y) return

    nextDirection = mapped

    if (!started) {
      started = true
      intervalId = setInterval(tick, TICK_MS)
    }
  }

  function tick() {
    if (gameOver) return

    direction = { ...nextDirection }

    const head = snake[0]
    const newHead = {
      x: head.x + direction.x,
      y: head.y + direction.y
    }

    // Wall collision
    if (
      newHead.x < 0 || newHead.x >= GRID_COLS ||
      newHead.y < 0 || newHead.y >= GRID_ROWS
    ) {
      endGame()
      return
    }

    // Self collision
    if (snake.some(seg => seg.x === newHead.x && seg.y === newHead.y)) {
      endGame()
      return
    }

    snake = [newHead, ...snake]

    if (newHead.x === food.x && newHead.y === food.y) {
      // FUTURE: score multipliers, combo chains, or level-up thresholds go here
      score += 1
      spawnFood()
      // tail stays — snake grows by one
    } else {
      snake = snake.slice(0, -1)
    }

    render()
  }

  function endGame() {
    gameOver = true
    clearInterval(intervalId)
    intervalId = null
    render()
    // FUTURE: submit score to a high score API or localStorage here
  }

  function restartGame() {
    initGame()
    render()
  }

  function render() {
    if (!ctx) return

    // Background
    ctx.fillStyle = COLOR_BG
    ctx.fillRect(0, 0, CANVAS_W, CANVAS_H)

    // Grid lines
    // FUTURE: toggle with a "show grid" user setting
    ctx.strokeStyle = COLOR_GRID
    ctx.lineWidth = 0.5
    for (let x = 0; x <= CANVAS_W; x += CELL_SIZE) {
      ctx.beginPath()
      ctx.moveTo(x, 0)
      ctx.lineTo(x, CANVAS_H)
      ctx.stroke()
    }
    for (let y = 0; y <= CANVAS_H; y += CELL_SIZE) {
      ctx.beginPath()
      ctx.moveTo(0, y)
      ctx.lineTo(CANVAS_W, y)
      ctx.stroke()
    }

    // Food — inset 2px to look like a dot, distinct from snake
    ctx.fillStyle = COLOR_FOOD
    ctx.fillRect(
      food.x * CELL_SIZE + 2,
      food.y * CELL_SIZE + 2,
      CELL_SIZE - 4,
      CELL_SIZE - 4
    )

    // Snake — inset 1px to show cell gaps; head is lighter
    snake.forEach((seg, i) => {
      ctx.fillStyle = i === 0 ? COLOR_HEAD : COLOR_SNAKE
      ctx.fillRect(
        seg.x * CELL_SIZE + 1,
        seg.y * CELL_SIZE + 1,
        CELL_SIZE - 2,
        CELL_SIZE - 2
      )
    })

    // Idle overlay before first keypress
    if (!started && !gameOver) {
      ctx.fillStyle = 'rgba(0, 0, 0, 0.55)'
      ctx.fillRect(0, 0, CANVAS_W, CANVAS_H)
      ctx.fillStyle = '#ffffff'
      ctx.font = 'bold 16px system-ui'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'middle'
      ctx.fillText('Press an arrow key to start', CANVAS_W / 2, CANVAS_H / 2)
    }

    // Game over overlay is rendered as HTML (see template) for accessible button
  }

  onMount(() => {
    ctx = canvas.getContext('2d')
    initGame()
    render()
    window.addEventListener('keydown', handleKeyDown)
  })

  onDestroy(() => {
    clearInterval(intervalId)
    window.removeEventListener('keydown', handleKeyDown)
  })
</script>

<div class="game-wrapper">
  <div class="score-bar">Score: <strong>{score}</strong></div>

  <div class="canvas-container">
    <canvas bind:this={canvas} width={CANVAS_W} height={CANVAS_H}></canvas>

    {#if gameOver}
      <div class="overlay">
        <p class="game-over-text">Game Over</p>
        <p class="final-score">Score: {score}</p>
        <button on:click={restartGame}>Play Again</button>
      </div>
    {/if}
  </div>
</div>

<style>
  .game-wrapper {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.75rem;
  }

  .score-bar {
    font-size: 1.25rem;
    min-width: 200px;
    text-align: center;
  }

  .canvas-container {
    position: relative;
  }

  canvas {
    display: block;
    border: 2px solid #4ade80;
    border-radius: 4px;
  }

  .overlay {
    position: absolute;
    inset: 0;
    background: rgba(0, 0, 0, 0.75);
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.75rem;
    border-radius: 4px;
  }

  .game-over-text {
    font-size: 2rem;
    font-weight: bold;
    color: #e94560;
  }

  .final-score {
    font-size: 1.1rem;
    color: #e0e0e0;
  }

  button {
    padding: 0.5rem 1.5rem;
    font-size: 1rem;
    background: #4ade80;
    color: #0f3460;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-weight: bold;
    transition: background 0.15s;
  }

  button:hover {
    background: #86efac;
  }
</style>
