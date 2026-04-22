<script>
  import { onMount, onDestroy, tick } from 'svelte'

  // --- Grid & timing ---
  const GRID_COLS    = 20
  const GRID_ROWS    = 20
  const BASE_TICK_MS = 150
  const MIN_TICK_MS  = 55

  // Speed increases every 3 points, bottoms out at MIN_TICK_MS
  $: tickMs = Math.max(MIN_TICK_MS, BASE_TICK_MS - Math.floor(score / 3) * 10)
  $: level  = Math.floor((BASE_TICK_MS - tickMs) / 10) + 1

  // CELL_SIZE is computed dynamically from the viewport at mount and on resize.
  // CANVAS_W/H are reactive and drive the <canvas> width/height attributes.
  let CELL_SIZE = 22
  $: CANVAS_W = GRID_COLS * CELL_SIZE
  $: CANVAS_H = GRID_ROWS * CELL_SIZE

  // --- Colors (RedSky brand) ---
  const COLOR_BG    = '#0c0c10'
  const COLOR_FOOD  = '#2563eb'
  const COLOR_SNAKE = '#cc1414'
  const COLOR_HEAD  = '#e84040'

  // --- Game state ---
  let snake = []
  let direction     = { x: 1, y: 0 }
  let nextDirection = { x: 1, y: 0 }
  let food    = { x: 0, y: 0 }
  let score   = 0
  let gameOver = false
  let started  = false

  // --- Canvas ---
  let canvas
  let ctx
  let bgCanvas
  let logoRef = null   // retained so resize can rebuild the background

  // --- Smooth rendering (fixed timestep + RAF interpolation) ---
  let rafId         = null
  let prevSnake     = []
  let accumulator   = 0
  let lastTimestamp = null

  // --- Sizing ---

  function computeCellSize() {
    // Reserve ~14% of height for header + score bar; at least 110px
    const reserved = Math.max(110, window.innerHeight * 0.14)
    const availH   = window.innerHeight - reserved
    const availW   = window.innerWidth  - 32   // 16px gutter each side
    const maxDim   = Math.min(availH, availW)
    CELL_SIZE = Math.max(18, Math.min(52, Math.floor(maxDim / GRID_COLS)))
  }

  let resizeTimer = null
  async function handleResize() {
    clearTimeout(resizeTimer)
    resizeTimer = setTimeout(async () => {
      computeCellSize()
      await tick()              // wait for Svelte to update canvas width/height attrs
      buildBackground(logoRef)  // rebuild offscreen bg at new size
    }, 80)
  }

  // --- Background ---

  // Pre-renders hex tiles + logo watermark to an offscreen canvas.
  // Called once at mount (and again on resize). Blits every frame for free.
  function buildBackground(logoImg) {
    bgCanvas        = document.createElement('canvas')
    bgCanvas.width  = CANVAS_W
    bgCanvas.height = CANVAS_H
    const bg = bgCanvas.getContext('2d')

    bg.fillStyle = COLOR_BG
    bg.fillRect(0, 0, CANVAS_W, CANVAS_H)

    // Flat-top hexagons: column spacing = 3R/2, row spacing = R√3,
    // odd columns offset vertically by R√3/2.
    const R    = Math.max(12, Math.round(CELL_SIZE * 0.72))
    const rowH = R * Math.sqrt(3)
    const colW = R * 1.5

    bg.strokeStyle = 'rgba(255,255,255,0.05)'
    bg.lineWidth   = 1

    for (let col = -1; col * colW < CANVAS_W + R * 2; col++) {
      for (let row = -1; row * rowH < CANVAS_H + rowH; row++) {
        const cx = col * colW
        const cy = row * rowH + (col % 2 !== 0 ? rowH / 2 : 0)

        bg.beginPath()
        for (let i = 0; i < 6; i++) {
          const angle = (Math.PI / 3) * i
          if (i === 0) bg.moveTo(cx + R * Math.cos(angle), cy + R * Math.sin(angle))
          else         bg.lineTo(cx + R * Math.cos(angle), cy + R * Math.sin(angle))
        }
        bg.closePath()
        bg.fillStyle = '#111118'
        bg.fill()
        bg.stroke()
      }
    }

    // RedSky "R" mark — centered watermark, very subtle
    if (logoImg) {
      const aspect = logoImg.naturalWidth / logoImg.naturalHeight
      const logoH  = CANVAS_H * 0.72
      const logoW  = logoH * aspect
      bg.globalAlpha = 0.045
      bg.drawImage(logoImg, (CANVAS_W - logoW) / 2, (CANVAS_H - logoH) / 2, logoW, logoH)
      bg.globalAlpha = 1
    }
  }

  // --- Game logic ---

  function initGame() {
    const midY = Math.floor(GRID_ROWS / 2)
    snake = [
      { x: 12, y: midY },
      { x: 11, y: midY },
      { x: 10, y: midY }
    ]
    direction     = { x: 1, y: 0 }
    nextDirection = { x: 1, y: 0 }
    score         = 0
    gameOver      = false
    started       = false
    prevSnake     = []
    accumulator   = 0
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
    } while (snake.some(s => s.x === pos.x && s.y === pos.y))
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
    if (mapped.x === -direction.x && mapped.y === -direction.y) return
    nextDirection = mapped
    if (!started) started = true
  }

  // Pure logic step — rendering is handled entirely by the RAF loop.
  function tickLogic() {
    if (gameOver) return
    direction = { ...nextDirection }

    const head    = snake[0]
    const newHead = { x: head.x + direction.x, y: head.y + direction.y }

    if (newHead.x < 0 || newHead.x >= GRID_COLS || newHead.y < 0 || newHead.y >= GRID_ROWS) {
      endGame(); return
    }
    if (snake.some(s => s.x === newHead.x && s.y === newHead.y)) {
      endGame(); return
    }

    snake = [newHead, ...snake]

    if (newHead.x === food.x && newHead.y === food.y) {
      // FUTURE: score multipliers, combo chains, or level-up thresholds go here
      score += 1
      spawnFood()
    } else {
      snake = snake.slice(0, -1)
    }
  }

  function endGame() {
    gameOver    = true
    accumulator = 0
    // FUTURE: submit score to a high score API or localStorage here
  }

  function restartGame() {
    initGame()
  }

  // --- RAF game loop ---

  function startGameLoop() {
    function loop(timestamp) {
      if (lastTimestamp !== null) {
        const dt = Math.min(timestamp - lastTimestamp, tickMs * 2)
        if (started && !gameOver) {
          accumulator += dt
          while (accumulator >= tickMs) {
            prevSnake = snake.map(s => ({ ...s }))
            tickLogic()
            accumulator -= tickMs
            if (gameOver) { accumulator = 0; break }
          }
        }
      }
      lastTimestamp = timestamp

      const t = (started && !gameOver && prevSnake.length > 0)
        ? accumulator / tickMs
        : 0
      render(t, timestamp)
      rafId = requestAnimationFrame(loop)
    }
    rafId = requestAnimationFrame(loop)
  }

  // --- Interpolation ---

  function lerp(a, b, t) { return a + (b - a) * t }

  // Returns pixel-space {x,y} for each segment, smoothly interpolated between ticks.
  // from = prevSnake[i] (where segment was); to = where it's heading.
  // At t=0: prevSnake positions. At t=1: snake positions.
  function getDrawSegments(t) {
    if (prevSnake.length === 0 || !started) {
      return snake.map(s => ({
        x: s.x * CELL_SIZE + CELL_SIZE / 2,
        y: s.y * CELL_SIZE + CELL_SIZE / 2
      }))
    }
    return snake.map((seg, i) => {
      const from = prevSnake[i] ?? seg
      const to   = i === 0 ? seg : prevSnake[i - 1]
      return {
        x: lerp(from.x, to.x, t) * CELL_SIZE + CELL_SIZE / 2,
        y: lerp(from.y, to.y, t) * CELL_SIZE + CELL_SIZE / 2
      }
    })
  }

  // --- Rendering ---

  function drawFood(now) {
    const cx = food.x * CELL_SIZE + CELL_SIZE / 2
    const cy = food.y * CELL_SIZE + CELL_SIZE / 2

    // Slow heartbeat pulse: 0 → 1 → 0 over ~3s
    const pulse  = 0.5 + 0.5 * Math.sin(now / 480)
    const glowR  = CELL_SIZE * (0.55 + 0.45 * pulse)
    const dotR   = CELL_SIZE * 0.15 + CELL_SIZE * 0.07 * pulse

    // Outer pulsing glow
    const glow = ctx.createRadialGradient(cx, cy, 0, cx, cy, glowR)
    glow.addColorStop(0, `rgba(37,99,235,${0.45 + 0.25 * pulse})`)
    glow.addColorStop(0.5, `rgba(37,99,235,${0.2 * pulse})`)
    glow.addColorStop(1, 'rgba(37,99,235,0)')
    ctx.beginPath()
    ctx.arc(cx, cy, glowR, 0, Math.PI * 2)
    ctx.fillStyle = glow
    ctx.fill()

    // Inner shiny dot
    const dot = ctx.createRadialGradient(cx - dotR * 0.3, cy - dotR * 0.3, 0, cx, cy, dotR)
    dot.addColorStop(0, '#bfdbfe')
    dot.addColorStop(1, COLOR_FOOD)
    ctx.beginPath()
    ctx.arc(cx, cy, dotR, 0, Math.PI * 2)
    ctx.fillStyle = dot
    ctx.fill()
  }

  function buildSnakePath(segments) {
    ctx.beginPath()
    segments.forEach(({ x, y }, i) => {
      if (i === 0) ctx.moveTo(x, y)
      else         ctx.lineTo(x, y)
    })
  }

  function drawSnakeBody(segments, now) {
    if (segments.length < 2) return

    ctx.lineCap  = 'round'
    ctx.lineJoin = 'round'

    // --- Ambient glow corona around the whole body ---
    buildSnakePath(segments)
    ctx.lineWidth   = CELL_SIZE + 12
    ctx.strokeStyle = 'rgba(204,20,20,0.05)'
    ctx.stroke()

    buildSnakePath(segments)
    ctx.lineWidth   = CELL_SIZE + 4
    ctx.strokeStyle = 'rgba(204,20,20,0.10)'
    ctx.stroke()

    // --- Shadow for depth ---
    buildSnakePath(segments)
    ctx.lineWidth   = CELL_SIZE - 1
    ctx.strokeStyle = 'rgba(0,0,0,0.55)'
    ctx.stroke()

    // --- Main body: gradient bright at head, dim at tail ---
    const head = segments[0], tail = segments[segments.length - 1]
    const grad = ctx.createLinearGradient(head.x, head.y, tail.x, tail.y)
    grad.addColorStop(0,   COLOR_SNAKE)
    grad.addColorStop(0.6, '#aa1010')
    grad.addColorStop(1,   'rgba(90,6,6,0.65)')
    buildSnakePath(segments)
    ctx.lineWidth   = CELL_SIZE - 5
    ctx.strokeStyle = grad
    ctx.stroke()

    // --- Specular highlight stripe ---
    buildSnakePath(segments)
    ctx.lineWidth   = (CELL_SIZE - 5) * 0.32
    ctx.strokeStyle = 'rgba(255,255,255,0.18)'
    ctx.stroke()
  }

  function drawSnakeHead(segments, now) {
    if (segments.length === 0) return

    const { x: cx, y: cy } = segments[0]
    const r = CELL_SIZE / 2

    // --- Pulsing energy corona ---
    const pulse   = 0.5 + 0.5 * Math.sin(now / 380)
    const coronaR = r * (1.65 + 0.55 * pulse)
    const corona  = ctx.createRadialGradient(cx, cy, r * 0.4, cx, cy, coronaR)
    corona.addColorStop(0, `rgba(232,64,64,${0.28 + 0.12 * pulse})`)
    corona.addColorStop(1, 'rgba(204,20,20,0)')
    ctx.beginPath()
    ctx.arc(cx, cy, coronaR, 0, Math.PI * 2)
    ctx.fillStyle = corona
    ctx.fill()

    // --- Drop shadow ---
    ctx.beginPath()
    ctx.arc(cx, cy, r + 2, 0, Math.PI * 2)
    ctx.fillStyle = 'rgba(0,0,0,0.55)'
    ctx.fill()

    // --- Head fill ---
    ctx.beginPath()
    ctx.arc(cx, cy, r, 0, Math.PI * 2)
    ctx.fillStyle = COLOR_HEAD
    ctx.fill()

    // --- 3-D shine ---
    const shine = ctx.createRadialGradient(cx - r * 0.3, cy - r * 0.35, 0, cx, cy, r)
    shine.addColorStop(0,   'rgba(255,255,255,0.40)')
    shine.addColorStop(0.5, 'rgba(255,255,255,0.06)')
    shine.addColorStop(1,   'rgba(255,255,255,0)')
    ctx.beginPath()
    ctx.arc(cx, cy, r, 0, Math.PI * 2)
    ctx.fillStyle = shine
    ctx.fill()

    // --- Eyes ---
    const dx = direction.x, dy = direction.y
    const px = -dy,         py = dx
    const fwd  = r * 0.42,  side = r * 0.46,  eyeR = r * 0.28

    for (const s of [1, -1]) {
      const ex = cx + dx * fwd + px * side * s
      const ey = cy + dy * fwd + py * side * s

      ctx.beginPath()
      ctx.arc(ex, ey, eyeR, 0, Math.PI * 2)
      ctx.fillStyle = 'white'
      ctx.fill()

      ctx.beginPath()
      ctx.arc(ex + dx * eyeR * 0.3, ey + dy * eyeR * 0.3, eyeR * 0.55, 0, Math.PI * 2)
      ctx.fillStyle = '#111'
      ctx.fill()
    }
  }

  function render(t = 0, now = 0) {
    if (!ctx || !bgCanvas) return

    ctx.drawImage(bgCanvas, 0, 0)

    drawFood(now)

    const segments = getDrawSegments(t)
    drawSnakeBody(segments, now)
    drawSnakeHead(segments, now)

    if (!started && !gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.62)'
      ctx.fillRect(0, 0, CANVAS_W, CANVAS_H)
      ctx.fillStyle    = '#ffffff'
      ctx.font         = `bold ${Math.round(CELL_SIZE * 0.72)}px system-ui`
      ctx.textAlign    = 'center'
      ctx.textBaseline = 'middle'
      ctx.fillText('Press an arrow key to start', CANVAS_W / 2, CANVAS_H / 2)
    }
  }

  // --- Lifecycle ---

  onMount(() => {
    ctx = canvas.getContext('2d')
    window.addEventListener('keydown', handleKeyDown)
    window.addEventListener('resize', handleResize)

    const img = new Image()
    const start = async (logoImg) => {
      logoRef = logoImg
      computeCellSize()
      await tick()             // let Svelte update canvas width/height attrs
      buildBackground(logoImg)
      initGame()
      startGameLoop()
    }
    img.onload  = () => start(img)
    img.onerror = () => start(null)
    img.src = '/logo.svg'
  })

  onDestroy(() => {
    if (rafId !== null) cancelAnimationFrame(rafId)
    clearTimeout(resizeTimer)
    window.removeEventListener('keydown', handleKeyDown)
    window.removeEventListener('resize', handleResize)
  })
</script>

<div class="game-wrapper">
  <div class="score-bar">
    Score: <strong>{score}</strong>
    <span class="level-badge" class:fast={level > 5}>Lvl {level}</span>
  </div>

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
    gap: 0.6rem;
  }

  .score-bar {
    font-size: 1.1rem;
    min-width: 160px;
    text-align: center;
    letter-spacing: 0.04em;
    display: flex;
    align-items: center;
    gap: 0.75rem;
  }

  .level-badge {
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    padding: 0.18rem 0.55rem;
    border-radius: 3px;
    background: rgba(204, 20, 20, 0.18);
    color: #cc1414;
    border: 1px solid rgba(204, 20, 20, 0.35);
    transition: background 0.3s, color 0.3s, border-color 0.3s;
  }

  .level-badge.fast {
    background: rgba(232, 64, 64, 0.28);
    color: #e84040;
    border-color: rgba(232, 64, 64, 0.6);
    box-shadow: 0 0 8px rgba(232, 64, 64, 0.3);
  }

  .canvas-container {
    position: relative;
  }

  canvas {
    display: block;
    border-radius: 6px;
    border: 1.5px solid #1e1e2a;
    box-shadow:
      0 0 0 1px rgba(204,20,20,0.08),
      0 0 40px rgba(204,20,20,0.18),
      0 0 80px rgba(204,20,20,0.08);
  }

  .overlay {
    position: absolute;
    inset: 0;
    background: rgba(0, 0, 0, 0.80);
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.75rem;
    border-radius: 6px;
  }

  .game-over-text {
    font-size: clamp(1.5rem, 4vw, 2.5rem);
    font-weight: 800;
    color: #e84040;
    letter-spacing: 0.06em;
    text-transform: uppercase;
  }

  .final-score {
    font-size: clamp(1rem, 2vw, 1.25rem);
    color: #e0e0e0;
    letter-spacing: 0.04em;
  }

  button {
    padding: 0.55rem 1.75rem;
    font-size: 1rem;
    background: #cc1414;
    color: #fff;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    transition: background 0.15s, box-shadow 0.15s;
  }

  button:hover {
    background: #e84040;
    box-shadow: 0 0 16px rgba(232,64,64,0.5);
  }
</style>
