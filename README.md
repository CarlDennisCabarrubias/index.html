<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Coin Runner</title>
  <style>
    /* =====================
       CODE MAPPING (inline)
       - HTML: Canvas + HUD + buttons
       - CSS: Layout + look & feel
       - JS: Game loop, events, collisions
       EVENTS USED
       - keydown / keyup: controls
       - click: Start/Restart
       - requestAnimationFrame: game loop
       CONTROLS
       - ←/A = move left
       - →/D = move right
       INSTRUCTIONS
       - Press Start, dodge red blocks, collect gold coins.
       - Survive and score as high as you can.
       ===================== */

    :root {
      --bg: #0f1226;
      --panel: #161a33;
      --text: #e9ecff;
      --accent: #7c9aff;
      --danger: #ff4d4d;
      --coin: #ffcc33;
      --player: #49e6a1;
    }

    * { box-sizing: border-box; }
    body {
      margin: 0;
      min-height: 100svh;
      display: grid;
      place-items: center;
      background: radial-gradient(1000px 600px at 50% -20%, #1d2250 0, var(--bg) 60%);
      color: var(--text);
      font-family: system-ui, -apple-system, Segoe UI, Roboto, Inter, Arial, sans-serif;
    }

    .wrap {
      width: min(92vw, 900px);
      display: grid;
      gap: 12px;
    }

    .hud {
      display: grid;
      grid-template-columns: 1fr auto auto;
      align-items: center;
      gap: 10px;
      background: color-mix(in oklab, var(--panel) 85%, black);
      padding: 10px 14px;
      border: 1px solid #262b58;
      border-radius: 14px;
      box-shadow: 0 6px 24px rgba(0,0,0,.3), inset 0 1px 0 rgba(255,255,255,.05);
    }

    .title { font-weight: 700; letter-spacing: .3px; }

    .stat {
      padding: 6px 10px;
      border-radius: 10px;
      background: #1b2247;
      border: 1px solid #2a3270;
      font-variant-numeric: tabular-nums;
    }

    button {
      appearance: none;
      border: none;
      background: linear-gradient(180deg, #8aa5ff, #6d8cff);
      color: #08123a;
      font-weight: 700;
      padding: 10px 14px;
      border-radius: 12px;
      cursor: pointer;
      box-shadow: 0 8px 18px rgba(108, 140, 255, .35);
    }
    button:disabled { opacity: .6; cursor: not-allowed; }

    canvas {
      width: 100%;
      height: auto;
      display: block;
      background: linear-gradient(180deg, #0c1030, #0a0d28 60%);
      border: 1px solid #262b58;
      border-radius: 16px;
      box-shadow: 0 12px 40px rgba(0,0,0,.45);
      image-rendering: pixelated;
    }

    .help {
      text-align: center;
      opacity: .9;
      font-size: .95rem;
    }
    kbd {
      background: #0f1540;
      border: 1px solid #2a3270;
      padding: 2px 6px;
      border-radius: 6px;
      box-shadow: inset 0 1px 0 rgba(255,255,255,.05);
      font-family: ui-monospace, Menlo, Consolas, monospace;
      font-size: .9em;
    }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="hud">
      <div class="title">🎮 Coin Runner</div>
      <div class="stat" id="score">Score: 0</div>
      <button id="startBtn">Start</button>
    </div>

    <canvas id="game" width="900" height="520" aria-label="Coin Runner game area"></canvas>

    <div class="help">
      Controls: <kbd>←</kbd>/<kbd>A</kbd> and <kbd>→</kbd>/<kbd>D</kbd> to move. Dodge red blocks, collect gold coins.
    </div>
  </div>

  <script>
    // =====================
    // GAME STATE & HELPERS
    // =====================
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    const scoreEl = document.getElementById('score');
    const startBtn = document.getElementById('startBtn');

    const W = canvas.width;
    const H = canvas.height;

    let keys = { left:false, right:false };
    let rafId = null;

    const rnd = (min, max) => Math.random() * (max - min) + min;

    // Objects
    const player = { x: W/2 - 18, y: H - 70, w: 36, h: 36, speed: 6 };
    let obstacles = []; // red blocks
    let coins = [];     // gold circles
    let t = 0;          // time accumulator
    let score = 0;
    let running = false;

    function reset() {
      player.x = W/2 - player.w/2;
      obstacles = [];
      coins = [];
      t = 0;
      score = 0;
      updateHUD();
    }

    function updateHUD(){
      scoreEl.textContent = `Score: ${score}`;
    }

    // =====================
    // INPUT EVENTS
    // =====================
    window.addEventListener('keydown', (e)=>{
      if(['ArrowLeft','a','A'].includes(e.key)) keys.left = true;
      if(['ArrowRight','d','D'].includes(e.key)) keys.right = true;
    });
    window.addEventListener('keyup', (e)=>{
      if(['ArrowLeft','a','A'].includes(e.key)) keys.left = false;
      if(['ArrowRight','d','D'].includes(e.key)) keys.right = false;
    });

    startBtn.addEventListener('click', ()=>{
      if (running) return;
      reset();
      running = true;
      startBtn.textContent = 'Playing…';
      startBtn.disabled = true;
      loop();
    });

    // =====================
    // GAME LOOP
    // =====================
    function loop(){
      rafId = requestAnimationFrame(loop);
      step();
      draw();
    }

    function step(){
      t += 1;

      // Spawn logic
      if (t % 40 === 0) { // obstacle every ~0.66s @60fps
        obstacles.push({ x: rnd(20, W-60), y: -30, w: rnd(30, 60), h: rnd(20, 40), vy: rnd(3,6) });
      }
      if (t % 55 === 0) { // coin every ~0.9s
        coins.push({ x: rnd(20, W-20), y: -18, r: 12, vy: rnd(2.5,5) });
      }

      // Player movement
      if (keys.left)  player.x -= player.speed;
      if (keys.right) player.x += player.speed;
      player.x = Math.max(10, Math.min(W - player.w - 10, player.x));

      // Update obstacles & coins
      obstacles.forEach(o => o.y += o.vy);
      coins.forEach(c => c.y += c.vy);

      // Remove offscreen
      obstacles = obstacles.filter(o => o.y < H + 80);
      coins = coins.filter(c => c.y < H + 40);

      // Collisions
      for (const o of obstacles) {
        if (aabb(player, o)) {
          gameOver();
          return;
        }
      }
      for (let i = coins.length - 1; i >= 0; i--) {
        const c = coins[i];
        if (rectCircleCollide(player, c)) {
          coins.splice(i, 1);
          score += 10;
          updateHUD();
        }
      }

      // Passive scoring over time
      if (t % 30 === 0) { score += 1; updateHUD(); }
    }

    function draw(){
      // Background grid
      ctx.clearRect(0,0,W,H);
      ctx.fillStyle = '#0a0f2e';
      ctx.fillRect(0,0,W,H);

      ctx.save();
      ctx.globalAlpha = .25;
      ctx.strokeStyle = '#2a3270';
      for (let x=0; x<W; x+=30){
        ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,H); ctx.stroke();
      }
      for (let y=0; y<H; y+=30){
        ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(W,y); ctx.stroke();
      }
      ctx.restore();

      // Player
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--player');
      roundRect(ctx, player.x, player.y, player.w, player.h, 8, true);

      // Obstacles
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--danger');
      obstacles.forEach(o => roundRect(ctx, o.x, o.y, o.w, o.h, 6, true));

      // Coins
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--coin');
      ctx.strokeStyle = '#f5e6a8';
      ctx.lineWidth = 2;
      coins.forEach(c => {
        ctx.beginPath();
        ctx.arc(c.x, c.y, c.r, 0, Math.PI*2);
        ctx.fill();
        ctx.stroke();
      });

      // HUD overlay hint when not running
      if (!running) {
        ctx.fillStyle = 'rgba(0,0,0,.55)';
        ctx.fillRect(0,0,W,H);
        ctx.fillStyle = 'white';
        ctx.textAlign = 'center';
        ctx.font = '700 32px system-ui, sans-serif';
        ctx.fillText('Press Start to Play', W/2, H/2 - 10);
        ctx.font = '14px system-ui, sans-serif';
        ctx.fillText('Use ←/A and →/D to move', W/2, H/2 + 20);
      }
    }

    // =====================
    // UTILITIES
    // =====================
    function aabb(r1, r2){
      return (
        r1.x < r2.x + r2.w &&
        r1.x + r1.w > r2.x &&
        r1.y < r2.y + r2.h &&
        r1.y + r1.h > r2.y
      );
    }

    function rectCircleCollide(rect, circ){
      const cx = Math.max(rect.x, Math.min(circ.x, rect.x + rect.w));
      const cy = Math.max(rect.y, Math.min(circ.y, rect.y + rect.h));
      const dx = circ.x - cx;
      const dy = circ.y - cy;
      return (dx*dx + dy*dy) <= (circ.r * circ.r);
    }

    function roundRect(ctx, x, y, w, h, r, fill){
      ctx.beginPath();
      ctx.moveTo(x+r, y);
      ctx.arcTo(x+w, y,   x+w, y+h, r);
      ctx.arcTo(x+w, y+h, x,   y+h, r);
      ctx.arcTo(x,   y+h, x,   y,   r);
      ctx.arcTo(x,   y,   x+w, y,   r);
      if (fill) ctx.fill(); else ctx.stroke();
    }

    function gameOver(){
      running = false;
      cancelAnimationFrame(rafId);
      startBtn.textContent = 'Restart';
      startBtn.disabled = false;

      // Dim and show text
      ctx.fillStyle = 'rgba(0,0,0,.6)';
      ctx.fillRect(0,0,W,H);
      ctx.fillStyle = 'white';
      ctx.textAlign = 'center';
      ctx.font = '700 40px system-ui, sans-serif';
      ctx.fillText('Game Over', W/2, H/2 - 20);
      ctx.font = '18px system-ui, sans-serif';
      ctx.fillText(`Final Score: ${score}`, W/2, H/2 + 10);
      ctx.font = '14px system-ui, sans-serif';
      ctx.fillText('Click Restart to play again', W/2, H/2 + 35);
    }

    // Initial paint
    draw();
  </script>
</body>
</html>
