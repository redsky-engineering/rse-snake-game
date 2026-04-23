<script>
  import { onMount, onDestroy, tick } from 'svelte'

  // --- Grid & timing ---
  // FUTURE: expose TICK_MS_BASE as a prop to implement difficulty levels
  const GRID_COLS      = 20
  const GRID_ROWS      = 20
  const TICK_MS_BASE   = 150
  let   tickMs         = TICK_MS_BASE   // mutated by boost / slow power-ups

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

  // Direction inputs are buffered so rapid sequences (e.g. up→left while moving
  // right) both land instead of one clobbering the other between ticks.
  let inputQueue = []

  // --- Obstacles ---
  // Static hazards that spawn every OBSTACLE_EVERY pellets eaten. Colliding
  // with one ends the game, just like hitting a wall.
  const OBSTACLE_EVERY    = 3    // one new obstacle per N pellets eaten
  const OBSTACLE_MIN_DIST = 3    // Manhattan distance from head when spawning
  let obstacles    = []
  let pelletsEaten = 0

  // --- Power-ups ---
  //   boost: temporary speed-up
  //   slow : temporary slow-down
  //   half : instantly halves snake length
  // Power-ups appear probabilistically after a normal pellet is eaten, linger
  // for POWERUP_TTL_TICKS, then vanish. Picking one up does NOT grow the snake,
  // but awards a score bonus and plays a distinct sound.
  const POWERUP_TYPES        = ['boost', 'slow', 'half']
  const POWERUP_SPAWN_CHANCE = 0.28
  const POWERUP_TTL_TICKS    = 45
  const POWERUP_BOOST_TICKS  = 50
  const POWERUP_SLOW_TICKS   = 50
  const POWERUP_BOOST_MULT   = 0.55   // smaller tickMs = faster snake
  const POWERUP_SLOW_MULT    = 1.75   // larger  tickMs = slower snake
  const POWERUP_SCORE_BONUS  = 3

  const POWERUP_COLORS = {
    boost: { core: '#facc15', mid: '#fde68a', symbol: '⚡' },
    slow:  { core: '#22d3ee', mid: '#a5f3fc', symbol: '❄' },
    half:  { core: '#22c55e', mid: '#86efac', symbol: '½' }
  }

  let powerUp      = null  // { x, y, type, ticksLeft }
  let activeEffect = null  // { type, ticksLeft } for 'boost' or 'slow'

  // --- Audio ---
  let audioCtx    = null
  let noiseBuffer = null

  function getAudioCtx() {
    if (!audioCtx) {
      audioCtx    = new AudioContext()
      noiseBuffer = null
    }
    if (audioCtx.state === 'suspended') audioCtx.resume()
    return audioCtx
  }

  function playEatSound() {
    const ac = getAudioCtx()
    const osc = ac.createOscillator()
    const gain = ac.createGain()
    osc.connect(gain)
    gain.connect(ac.destination)
    osc.frequency.setValueAtTime(520, ac.currentTime)
    osc.frequency.exponentialRampToValueAtTime(880, ac.currentTime + 0.08)
    gain.gain.setValueAtTime(0.18, ac.currentTime)
    gain.gain.exponentialRampToValueAtTime(0.001, ac.currentTime + 0.18)
    osc.start(ac.currentTime)
    osc.stop(ac.currentTime + 0.18)
  }

  function playGameOverSound() {
    const ac = getAudioCtx()
    const now = ac.currentTime
    ;[440, 330, 200].forEach((freq, i) => {
      const osc = ac.createOscillator()
      const gain = ac.createGain()
      osc.connect(gain)
      gain.connect(ac.destination)
      osc.type = 'sawtooth'
      osc.frequency.setValueAtTime(freq, now + i * 0.13)
      osc.frequency.exponentialRampToValueAtTime(freq * 0.7, now + i * 0.13 + 0.22)
      gain.gain.setValueAtTime(0.14, now + i * 0.13)
      gain.gain.exponentialRampToValueAtTime(0.001, now + i * 0.13 + 0.28)
      osc.start(now + i * 0.13)
      osc.stop(now + i * 0.13 + 0.28)
    })
  }

  function playPowerUpSound(type) {
    const ac = getAudioCtx()
    const t0 = ac.currentTime

    if (type === 'boost') {
      // Fast ascending triad
      for (let i = 0; i < 3; i++) {
        const osc = ac.createOscillator()
        const g   = ac.createGain()
        osc.connect(g); g.connect(ac.destination)
        osc.type = 'triangle'
        osc.frequency.setValueAtTime(660 + i * 220, t0 + i * 0.055)
        g.gain.setValueAtTime(0.2, t0 + i * 0.055)
        g.gain.exponentialRampToValueAtTime(0.001, t0 + i * 0.055 + 0.14)
        osc.start(t0 + i * 0.055)
        osc.stop(t0 + i * 0.055 + 0.16)
      }
    } else if (type === 'slow') {
      // Descending sine swoop
      const osc = ac.createOscillator()
      const g   = ac.createGain()
      osc.connect(g); g.connect(ac.destination)
      osc.type = 'sine'
      osc.frequency.setValueAtTime(720, t0)
      osc.frequency.exponentialRampToValueAtTime(180, t0 + 0.45)
      g.gain.setValueAtTime(0.22, t0)
      g.gain.exponentialRampToValueAtTime(0.001, t0 + 0.5)
      osc.start(t0); osc.stop(t0 + 0.55)
    } else if (type === 'half') {
      // Two scissor-like square clicks + bandpassed noise
      for (const off of [0, 0.1]) {
        const osc = ac.createOscillator()
        const g   = ac.createGain()
        osc.connect(g); g.connect(ac.destination)
        osc.type = 'square'
        osc.frequency.setValueAtTime(1200, t0 + off)
        osc.frequency.exponentialRampToValueAtTime(520, t0 + off + 0.06)
        g.gain.setValueAtTime(0.18, t0 + off)
        g.gain.exponentialRampToValueAtTime(0.001, t0 + off + 0.08)
        osc.start(t0 + off); osc.stop(t0 + off + 0.09)
      }
      const src  = ac.createBufferSource()
      src.buffer = getNoiseBuf()
      const filt = ac.createBiquadFilter()
      filt.type            = 'bandpass'
      filt.frequency.value = 3000
      const g = ac.createGain()
      g.gain.setValueAtTime(0.12, t0)
      g.gain.exponentialRampToValueAtTime(0.001, t0 + 0.18)
      src.connect(filt); filt.connect(g); g.connect(ac.destination)
      src.start(t0); src.stop(t0 + 0.2)
    }
  }

  // --- Music ---
  let musicPlaying     = false
  let musicStep        = 0
  let musicNextTime    = 0
  let musicSchedulerId = null

  const MUSIC_BPM   = 130
  const STEP_S      = 60 / MUSIC_BPM / 2   // eighth-note duration
  const LOOK_AHEAD  = 0.1                   // seconds to schedule ahead
  const SCHED_MS    = 50                    // scheduler poll interval

  // A-minor pentatonic, 4-bar loop (32 × eighth notes ≈ 7.4 s)
  const BASS_PAT = [
    45,null,52,null, 45,null,45,null,
    43,null,45,null, 52,null,45,null,
    45,null,52,null, 50,45,  52,null,
    45,null,45,null, 45,50,  52,null
  ]
  const MELODY_PAT = [
    69,null,null,72, 76,74, null,72,
    69,null,67, 69, null,64, 67,null,
    72,null,null,76, 74,72, null,69,
    67,69,  null,72, 69,null,67,null
  ]
  const KICK_PAT = [
    1,0,0,0, 1,0,0,0, 1,0,0,0, 1,0,0,0,
    1,0,0,0, 1,0,0,0, 1,0,0,0, 1,0,1,0
  ]
  const HAT_PAT = [
    0,0,1,0, 0,0,1,0, 0,0,1,0, 0,0,1,0,
    0,0,1,0, 0,0,1,0, 0,0,1,0, 0,0,1,0
  ]

  function midiHz(midi) { return 440 * Math.pow(2, (midi - 69) / 12) }

  function getNoiseBuf() {
    if (noiseBuffer) return noiseBuffer
    const ac  = getAudioCtx()
    const len = Math.floor(ac.sampleRate * 0.05)
    noiseBuffer = ac.createBuffer(1, len, ac.sampleRate)
    const d = noiseBuffer.getChannelData(0)
    for (let i = 0; i < len; i++) d[i] = Math.random() * 2 - 1
    return noiseBuffer
  }

  function schedBass(midi, t) {
    const ac   = getAudioCtx()
    const osc  = ac.createOscillator()
    const filt = ac.createBiquadFilter()
    const g    = ac.createGain()
    osc.type = 'sawtooth'
    osc.frequency.value  = midiHz(midi)
    filt.type            = 'lowpass'
    filt.frequency.value = 700
    g.gain.setValueAtTime(0.15, t)
    g.gain.exponentialRampToValueAtTime(0.001, t + STEP_S * 0.85)
    osc.connect(filt); filt.connect(g); g.connect(ac.destination)
    osc.start(t); osc.stop(t + STEP_S)
  }

  function schedMelody(midi, t) {
    const ac  = getAudioCtx()
    const osc = ac.createOscillator()
    const g   = ac.createGain()
    osc.type = 'square'
    osc.frequency.value = midiHz(midi)
    g.gain.setValueAtTime(0.04, t)
    g.gain.exponentialRampToValueAtTime(0.001, t + STEP_S * 1.9)
    osc.connect(g); g.connect(ac.destination)
    osc.start(t); osc.stop(t + STEP_S * 2)
  }

  function schedKick(t) {
    const ac  = getAudioCtx()
    const osc = ac.createOscillator()
    const g   = ac.createGain()
    osc.frequency.setValueAtTime(160, t)
    osc.frequency.exponentialRampToValueAtTime(40, t + 0.08)
    g.gain.setValueAtTime(0.28, t)
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.18)
    osc.connect(g); g.connect(ac.destination)
    osc.start(t); osc.stop(t + 0.2)
  }

  function schedHat(t) {
    const ac   = getAudioCtx()
    const src  = ac.createBufferSource()
    src.buffer = getNoiseBuf()
    const filt = ac.createBiquadFilter()
    filt.type            = 'highpass'
    filt.frequency.value = 7000
    const g = ac.createGain()
    g.gain.setValueAtTime(0.06, t)
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.04)
    src.connect(filt); filt.connect(g); g.connect(ac.destination)
    src.start(t); src.stop(t + 0.05)
  }

  function musicSchedule() {
    if (!musicPlaying) return
    const ac = getAudioCtx()
    while (musicNextTime < ac.currentTime + LOOK_AHEAD) {
      const i = musicStep % BASS_PAT.length
      if (BASS_PAT[i]   != null) schedBass(BASS_PAT[i], musicNextTime)
      if (MELODY_PAT[i] != null) schedMelody(MELODY_PAT[i], musicNextTime)
      if (KICK_PAT[i])           schedKick(musicNextTime)
      if (HAT_PAT[i])            schedHat(musicNextTime)
      musicNextTime += STEP_S
      musicStep++
    }
    musicSchedulerId = setTimeout(musicSchedule, SCHED_MS)
  }

  function startMusic() {
    if (musicPlaying) return
    const ac      = getAudioCtx()
    musicPlaying  = true
    musicStep     = 0
    musicNextTime = ac.currentTime + 0.05
    musicSchedule()
  }

  function stopMusic() {
    musicPlaying = false
    if (musicSchedulerId !== null) {
      clearTimeout(musicSchedulerId)
      musicSchedulerId = null
    }
  }

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
    powerUp       = null
    activeEffect  = null
    tickMs        = TICK_MS_BASE
    inputQueue    = []
    obstacles     = []
    pelletsEaten  = 0
    spawnFood()
  }

  function spawnFood() {
    let pos
    do {
      pos = {
        x: Math.floor(Math.random() * GRID_COLS),
        y: Math.floor(Math.random() * GRID_ROWS)
      }
    } while (
      snake.some(s => s.x === pos.x && s.y === pos.y) ||
      (powerUp && pos.x === powerUp.x && pos.y === powerUp.y) ||
      obstacles.some(o => o.x === pos.x && o.y === pos.y)
    )
    food = pos
  }

  function maybeSpawnPowerUp() {
    if (powerUp) return
    if (Math.random() > POWERUP_SPAWN_CHANCE) return
    const type = POWERUP_TYPES[Math.floor(Math.random() * POWERUP_TYPES.length)]
    let pos, attempts = 0
    do {
      pos = {
        x: Math.floor(Math.random() * GRID_COLS),
        y: Math.floor(Math.random() * GRID_ROWS)
      }
      if (++attempts > 50) return
    } while (
      snake.some(s => s.x === pos.x && s.y === pos.y) ||
      (pos.x === food.x && pos.y === food.y) ||
      obstacles.some(o => o.x === pos.x && o.y === pos.y)
    )
    powerUp = { x: pos.x, y: pos.y, type, ticksLeft: POWERUP_TTL_TICKS }
  }

  function spawnObstacle() {
    const head = snake[0]
    let pos, attempts = 0
    do {
      pos = {
        x: Math.floor(Math.random() * GRID_COLS),
        y: Math.floor(Math.random() * GRID_ROWS)
      }
      if (++attempts > 100) return   // board too crowded; skip this round
    } while (
      snake.some(s => s.x === pos.x && s.y === pos.y) ||
      (pos.x === food.x && pos.y === food.y) ||
      (powerUp && pos.x === powerUp.x && pos.y === powerUp.y) ||
      obstacles.some(o => o.x === pos.x && o.y === pos.y) ||
      (Math.abs(pos.x - head.x) + Math.abs(pos.y - head.y) < OBSTACLE_MIN_DIST)
    )
    obstacles = [...obstacles, pos]
  }

  function applyPowerUp(type) {
    if (type === 'boost') {
      activeEffect = { type: 'boost', ticksLeft: POWERUP_BOOST_TICKS }
      tickMs = TICK_MS_BASE * POWERUP_BOOST_MULT
    } else if (type === 'slow') {
      activeEffect = { type: 'slow', ticksLeft: POWERUP_SLOW_TICKS }
      tickMs = TICK_MS_BASE * POWERUP_SLOW_MULT
    } else if (type === 'half') {
      const newLen = Math.max(3, Math.floor(snake.length / 2))
      snake = snake.slice(0, newLen)
      if (prevSnake.length > newLen) prevSnake = prevSnake.slice(0, newLen)
    }
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

    // Validate against the *latest* intended direction (last queued, or current
    // nextDirection if the queue is empty). Without this, pressing up→left
    // while moving right would reject the "left" press against the stale
    // `direction` even though the snake is about to head up.
    const lastDir = inputQueue.length > 0
      ? inputQueue[inputQueue.length - 1]
      : nextDirection
    if (mapped.x === -lastDir.x && mapped.y === -lastDir.y) return  // 180° reversal
    if (mapped.x ===  lastDir.x && mapped.y ===  lastDir.y) return  // duplicate

    inputQueue.push(mapped)
    if (inputQueue.length > 2) inputQueue.shift()

    // Eager tick: fast-forward the accumulator so the pending turn commits
    // within ~40% of a tick instead of waiting out the full window. Makes
    // controls feel snappy without actually changing game speed.
    if (started && !gameOver) {
      const minAccum = tickMs * 0.6
      if (accumulator < minAccum) accumulator = minAccum
    }

    if (!started) { started = true; startMusic() }
  }

  // Pure logic step — rendering is handled entirely by the RAF loop.
  function tickLogic() {
    if (gameOver) return

    // Commit one queued direction change per tick (FIFO). Safety check against
    // current direction so we never reverse into the body.
    if (inputQueue.length > 0) {
      const next = inputQueue.shift()
      if (!(next.x === -direction.x && next.y === -direction.y)) {
        nextDirection = next
      }
    }
    direction = { ...nextDirection }

    // Age timed effect (boost / slow) and restore base speed when it ends.
    if (activeEffect) {
      activeEffect = { ...activeEffect, ticksLeft: activeEffect.ticksLeft - 1 }
      if (activeEffect.ticksLeft <= 0) {
        activeEffect = null
        tickMs = TICK_MS_BASE
      }
    }
    // Age any power-up waiting on the board; expire when its TTL runs out.
    if (powerUp) {
      powerUp = { ...powerUp, ticksLeft: powerUp.ticksLeft - 1 }
      if (powerUp.ticksLeft <= 0) powerUp = null
    }

    const head    = snake[0]
    const newHead = { x: head.x + direction.x, y: head.y + direction.y }

    if (newHead.x < 0 || newHead.x >= GRID_COLS || newHead.y < 0 || newHead.y >= GRID_ROWS) {
      endGame(); return
    }
    if (snake.some(s => s.x === newHead.x && s.y === newHead.y)) {
      endGame(); return
    }
    if (obstacles.some(o => o.x === newHead.x && o.y === newHead.y)) {
      endGame(); return
    }

    snake = [newHead, ...snake]

    if (newHead.x === food.x && newHead.y === food.y) {
      score += 1
      pelletsEaten += 1
      playEatSound()
      spawnFood()
      maybeSpawnPowerUp()
      if (pelletsEaten % OBSTACLE_EVERY === 0) spawnObstacle()
    } else {
      snake = snake.slice(0, -1)
    }

    // Power-up pickup — does NOT grow the snake, but triggers its effect.
    if (powerUp && newHead.x === powerUp.x && newHead.y === powerUp.y) {
      const type = powerUp.type
      powerUp = null
      score += POWERUP_SCORE_BONUS
      playPowerUpSound(type)
      applyPowerUp(type)
    }
  }

  function endGame() {
    gameOver    = true
    accumulator = 0
    stopMusic()
    playGameOverSound()
    // FUTURE: submit score to a high score API or localStorage here
  }

  function restartGame() {
    stopMusic()
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

  function drawObstacles(now) {
    if (obstacles.length === 0) return
    const rBase = CELL_SIZE * 0.42
    const spikes = 6

    for (const o of obstacles) {
      const cx = o.x * CELL_SIZE + CELL_SIZE / 2
      const cy = o.y * CELL_SIZE + CELL_SIZE / 2

      // Faint red warning halo
      const halo = ctx.createRadialGradient(cx, cy, 0, cx, cy, rBase * 1.7)
      halo.addColorStop(0, 'rgba(220,38,38,0.22)')
      halo.addColorStop(1, 'rgba(220,38,38,0)')
      ctx.beginPath()
      ctx.arc(cx, cy, rBase * 1.7, 0, Math.PI * 2)
      ctx.fillStyle = halo
      ctx.fill()

      // Jagged 6-point star (caltrop)
      ctx.beginPath()
      for (let i = 0; i < spikes * 2; i++) {
        const angle = (i / (spikes * 2)) * Math.PI * 2 - Math.PI / 2
        const rad = i % 2 === 0 ? rBase : rBase * 0.48
        const x = cx + Math.cos(angle) * rad
        const y = cy + Math.sin(angle) * rad
        if (i === 0) ctx.moveTo(x, y)
        else         ctx.lineTo(x, y)
      }
      ctx.closePath()

      const body = ctx.createRadialGradient(cx - rBase * 0.3, cy - rBase * 0.3, 0, cx, cy, rBase)
      body.addColorStop(0, '#4b5563')
      body.addColorStop(1, '#111827')
      ctx.fillStyle = body
      ctx.fill()

      ctx.strokeStyle = 'rgba(220,38,38,0.7)'
      ctx.lineWidth   = 1.5
      ctx.stroke()
    }
  }

  function hexToRgba(hex, a) {
    const n = parseInt(hex.slice(1), 16)
    const r = (n >> 16) & 255
    const g = (n >> 8) & 255
    const b = n & 255
    return `rgba(${r},${g},${b},${a})`
  }

  function drawPowerUp(now) {
    if (!powerUp) return
    const style = POWERUP_COLORS[powerUp.type]
    const cx = powerUp.x * CELL_SIZE + CELL_SIZE / 2
    const cy = powerUp.y * CELL_SIZE + CELL_SIZE / 2

    const pulse = 0.5 + 0.5 * Math.sin(now / 200)
    const lowTtl = powerUp.ticksLeft < 12
    const blink  = lowTtl ? 0.4 + 0.6 * Math.abs(Math.sin(now / 70)) : 1

    const glowR = CELL_SIZE * (0.75 + 0.35 * pulse)
    const dotR  = CELL_SIZE * 0.38

    ctx.save()
    ctx.globalAlpha = blink

    // Outer pulsing glow (type-tinted)
    const glow = ctx.createRadialGradient(cx, cy, 0, cx, cy, glowR)
    glow.addColorStop(0,    hexToRgba(style.core, 0.55 + 0.25 * pulse))
    glow.addColorStop(0.55, hexToRgba(style.core, 0.2 * pulse))
    glow.addColorStop(1,    hexToRgba(style.core, 0))
    ctx.beginPath()
    ctx.arc(cx, cy, glowR, 0, Math.PI * 2)
    ctx.fillStyle = glow
    ctx.fill()

    // Core disc with shiny highlight
    const disc = ctx.createRadialGradient(cx - dotR * 0.3, cy - dotR * 0.3, 0, cx, cy, dotR)
    disc.addColorStop(0, style.mid)
    disc.addColorStop(1, style.core)
    ctx.beginPath()
    ctx.arc(cx, cy, dotR, 0, Math.PI * 2)
    ctx.fillStyle = disc
    ctx.fill()

    // Symbol glyph
    ctx.font         = `bold ${Math.round(CELL_SIZE * 0.5)}px system-ui, -apple-system, "Segoe UI Symbol"`
    ctx.textAlign    = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillStyle    = '#0c0c10'
    ctx.fillText(style.symbol, cx, cy + 1)

    ctx.restore()
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

    drawObstacles(now)
    drawFood(now)
    drawPowerUp(now)

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
    stopMusic()
    if (audioCtx) audioCtx.close()
  })
</script>

<div class="game-wrapper">
  <div class="score-bar">
    <span>Score: <strong>{score}</strong></span>
    {#if activeEffect}
      <span class="effect-badge effect-{activeEffect.type}">
        {#if activeEffect.type === 'boost'}⚡ BOOST{:else}❄ SLOW{/if}
        <small>{(activeEffect.ticksLeft * tickMs / 1000).toFixed(1)}s</small>
      </span>
    {/if}
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
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.75rem;
    flex-wrap: wrap;
    font-size: 1.1rem;
    min-width: 160px;
    text-align: center;
    letter-spacing: 0.04em;
  }

  .effect-badge {
    display: inline-flex;
    align-items: center;
    gap: 0.35rem;
    padding: 0.2rem 0.65rem;
    border-radius: 999px;
    font-size: 0.85rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    border: 1px solid transparent;
    animation: effect-pulse 1.2s ease-in-out infinite;
  }

  .effect-badge small {
    font-weight: 600;
    opacity: 0.8;
    font-size: 0.72rem;
    letter-spacing: 0;
  }

  .effect-boost {
    background: rgba(250, 204, 21, 0.14);
    color: #facc15;
    border-color: rgba(250, 204, 21, 0.45);
  }

  .effect-slow {
    background: rgba(34, 211, 238, 0.14);
    color: #22d3ee;
    border-color: rgba(34, 211, 238, 0.45);
  }

  @keyframes effect-pulse {
    0%, 100% { opacity: 1;    transform: scale(1); }
    50%      { opacity: 0.75; transform: scale(0.97); }
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
