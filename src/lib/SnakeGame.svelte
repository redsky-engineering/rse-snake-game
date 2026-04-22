<script>
  import { onMount, onDestroy } from 'svelte'

  // --- Grid & timing ---
  // FUTURE: expose TICK_MS as a prop to implement difficulty levels
  const GRID_COLS = 20
  const GRID_ROWS = 20
  const CELL_SIZE = 22
  const CANVAS_W = GRID_COLS * CELL_SIZE
  const CANVAS_H = GRID_ROWS * CELL_SIZE
  const TICK_MS = 150

  // --- Colors ---
  const COLOR_BG    = '#18181c'
  const COLOR_FOOD  = '#e94560'
  const COLOR_SNAKE = '#4ade80'
  const COLOR_HEAD  = '#a3f0c0'

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
  let bgCanvas

  // --- Smooth rendering state ---
  // Game logic ticks at TICK_MS. requestAnimationFrame renders at 60fps.
  // Between ticks we interpolate each segment's position using `accumulator / TICK_MS`.
  let rafId = null
  let prevSnake = []      // snake positions at the start of the current tick interval
  let accumulator = 0     // ms elapsed since the last game logic tick
  let lastTimestamp = null

  // Builds a hex-tiled background once on mount — blit per frame instead of redrawing
  function buildBackground() {
    bgCanvas = document.createElement('canvas')
    bgCanvas.width = CANVAS_W
    bgCanvas.height = CANVAS_H
    const bg = bgCanvas.getContext('2d')

    bg.fillStyle = COLOR_BG
    bg.fillRect(0, 0, CANVAS_W, CANVAS_H)

    // Flat-top hexagons. R = circumradius (center → vertex).
    // Column spacing = 3R/2. Row spacing = R√3. Odd columns offset by R√3/2.
    const R = 16
    const rowH = R * Math.sqrt(3)
    const colW = R * 1.5

    bg.strokeStyle = 'rgba(255,255,255,0.07)'
    bg.lineWidth = 1

    for (let col = -1; col * colW < CANVAS_W + R * 2; col++) {
      for (let row = -1; row * rowH < CANVAS_H + rowH; row++) {
        const cx = col * colW
        const cy = row * rowH + (col % 2 !== 0 ? rowH / 2 : 0)

        bg.beginPath()
        for (let i = 0; i < 6; i++) {
          const angle = (Math.PI / 3) * i
          const vx = cx + R * Math.cos(angle)
          const vy = cy + R * Math.sin(angle)
          if (i === 0) bg.moveTo(vx, vy)
          else bg.lineTo(vx, vy)
        }
        bg.closePath()
        bg.fillStyle = '#1c1c22'
        bg.fill()
        bg.stroke()
      }
    }
  }

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
    prevSnake = []
    accumulator = 0
    lastTimestamp = null
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

    // RAF loop is always running; just flip the flag so it starts accumulating
    if (!started) started = true
  }

  // Pure game logic — does not call render(). The RAF loop handles all rendering.
  function tickLogic() {
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
  }

  function endGame() {
    gameOver = true
    accumulator = 0
    // FUTURE: submit score to a high score API or localStorage here
  }

  function restartGame() {
    initGame()
  }

  // --- RAF game loop (fixed timestep + interpolated rendering) ---

  function startGameLoop() {
    function loop(timestamp) {
      if (lastTimestamp !== null) {
        // Cap dt to avoid a "spiral of death" after the tab is backgrounded
        const dt = Math.min(timestamp - lastTimestamp, TICK_MS * 2)

        if (started && !gameOver) {
          accumulator += dt
          while (accumulator >= TICK_MS) {
            prevSnake = snake.map(s => ({ ...s }))
            tickLogic()
            accumulator -= TICK_MS
            if (gameOver) { accumulator = 0; break }
          }
        }
      }

      lastTimestamp = timestamp

      // t ∈ [0,1): how far we are between the last tick and the next
      const t = (started && !gameOver && prevSnake.length > 0)
        ? accumulator / TICK_MS
        : 0
      render(t)

      rafId = requestAnimationFrame(loop)
    }

    rafId = requestAnimationFrame(loop)
  }

  // --- Interpolation ---

  function lerp(a, b, t) {
    return a + (b - a) * t
  }

  // Returns pixel-space {x,y} for each segment, smoothly interpolated between ticks.
  //
  // The formula per segment i:
  //   from = prevSnake[i] (where it was at tick start; falls back to current pos for a grown tail)
  //   to   = snake[0]        (head travels to its new cell)
  //        = prevSnake[i-1]  (body/tail follows the segment ahead of it)
  //
  // At t=0 this reproduces prevSnake. At t=1 it reproduces snake. In between it glides.
  function getDrawSegments(t) {
    if (prevSnake.length === 0 || !started) {
      return snake.map(seg => ({
        x: seg.x * CELL_SIZE + CELL_SIZE / 2,
        y: seg.y * CELL_SIZE + CELL_SIZE / 2
      }))
    }

    return snake.map((seg, i) => {
      const from = prevSnake[i] ?? seg          // grown tail has no prev → stay put
      const to   = i === 0 ? seg : prevSnake[i - 1]
      return {
        x: lerp(from.x, to.x, t) * CELL_SIZE + CELL_SIZE / 2,
        y: lerp(from.y, to.y, t) * CELL_SIZE + CELL_SIZE / 2
      }
    })
  }

  // --- Rendering helpers ---

  function drawFood() {
    const cx = food.x * CELL_SIZE + CELL_SIZE / 2
    const cy = food.y * CELL_SIZE + CELL_SIZE / 2

    // Soft outer glow
    const glow = ctx.createRadialGradient(cx, cy, 0, cx, cy, CELL_SIZE * 0.8)
    glow.addColorStop(0, 'rgba(233,69,96,0.55)')
    glow.addColorStop(1, 'rgba(233,69,96,0)')
    ctx.beginPath()
    ctx.arc(cx, cy, CELL_SIZE * 0.8, 0, Math.PI * 2)
    ctx.fillStyle = glow
    ctx.fill()

    // Shiny inner dot
    const dot = ctx.createRadialGradient(cx - 1.5, cy - 1.5, 0, cx, cy, 5)
    dot.addColorStop(0, '#ff8fa3')
    dot.addColorStop(1, COLOR_FOOD)
    ctx.beginPath()
    ctx.arc(cx, cy, 5, 0, Math.PI * 2)
    ctx.fillStyle = dot
    ctx.fill()
  }

  function buildSnakePath(segments) {
    ctx.beginPath()
    segments.forEach(({ x, y }, i) => {
      if (i === 0) ctx.moveTo(x, y)
      else ctx.lineTo(x, y)
    })
  }

  function drawSnakeBody(segments) {
    if (segments.length < 2) return

    ctx.lineCap = 'round'
    ctx.lineJoin = 'round'

    // Dark shadow for depth
    buildSnakePath(segments)
    ctx.lineWidth = CELL_SIZE - 1
    ctx.strokeStyle = 'rgba(0,0,0,0.5)'
    ctx.stroke()

    // Main body tube
    buildSnakePath(segments)
    ctx.lineWidth = CELL_SIZE - 5
    ctx.strokeStyle = COLOR_SNAKE
    ctx.stroke()

    // Specular highlight along the top edge
    buildSnakePath(segments)
    ctx.lineWidth = (CELL_SIZE - 5) * 0.35
    ctx.strokeStyle = 'rgba(255,255,255,0.2)'
    ctx.stroke()
  }

  function drawSnakeHead(segments) {
    if (segments.length === 0) return

    const { x: cx, y: cy } = segments[0]
    const r = CELL_SIZE / 2

    // Shadow
    ctx.beginPath()
    ctx.arc(cx, cy, r + 1.5, 0, Math.PI * 2)
    ctx.fillStyle = 'rgba(0,0,0,0.5)'
    ctx.fill()

    // Head fill
    ctx.beginPath()
    ctx.arc(cx, cy, r, 0, Math.PI * 2)
    ctx.fillStyle = COLOR_HEAD
    ctx.fill()

    // Radial highlight (makes it look round/3D)
    const shine = ctx.createRadialGradient(cx - r * 0.3, cy - r * 0.35, 0, cx, cy, r)
    shine.addColorStop(0, 'rgba(255,255,255,0.38)')
    shine.addColorStop(0.6, 'rgba(255,255,255,0.05)')
    shine.addColorStop(1, 'rgba(255,255,255,0)')
    ctx.beginPath()
    ctx.arc(cx, cy, r, 0, Math.PI * 2)
    ctx.fillStyle = shine
    ctx.fill()

    // Eyes — positioned forward in the movement direction, one on each side
    const dx = direction.x, dy = direction.y
    const px = -dy,          py = dx   // perpendicular axis
    const fwd  = r * 0.42
    const side = r * 0.46
    const eyeR = r * 0.28

    for (const s of [1, -1]) {
      const ex = cx + dx * fwd + px * side * s
      const ey = cy + dy * fwd + py * side * s

      // Sclera
      ctx.beginPath()
      ctx.arc(ex, ey, eyeR, 0, Math.PI * 2)
      ctx.fillStyle = 'white'
      ctx.fill()

      // Pupil (shifted slightly toward travel direction)
      ctx.beginPath()
      ctx.arc(ex + dx * eyeR * 0.3, ey + dy * eyeR * 0.3, eyeR * 0.55, 0, Math.PI * 2)
      ctx.fillStyle = '#111'
      ctx.fill()
    }
  }

  function render(t = 0) {
    if (!ctx) return

    // Blit pre-rendered hex background
    ctx.drawImage(bgCanvas, 0, 0)

    drawFood()

    const segments = getDrawSegments(t)
    drawSnakeBody(segments)
    drawSnakeHead(segments)

    // Idle overlay (before first keypress)
    if (!started && !gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.6)'
      ctx.fillRect(0, 0, CANVAS_W, CANVAS_H)
      ctx.fillStyle = '#ffffff'
      ctx.font = 'bold 16px system-ui'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'middle'
      ctx.fillText('Press an arrow key to start', CANVAS_W / 2, CANVAS_H / 2)
    }

    // Game over is an HTML overlay (see template) — native button + accessibility
  }

  onMount(() => {
    ctx = canvas.getContext('2d')
    buildBackground()
    initGame()
    startGameLoop()
    window.addEventListener('keydown', handleKeyDown)
  })

  onDestroy(() => {
    if (rafId !== null) cancelAnimationFrame(rafId)
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
    border: 2px solid #2a2a32;
    border-radius: 6px;
    box-shadow: 0 0 40px rgba(74,222,128,0.08);
  }

  .overlay {
    position: absolute;
    inset: 0;
    background: rgba(0, 0, 0, 0.78);
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.75rem;
    border-radius: 6px;
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
    color: #0f1a0f;
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
