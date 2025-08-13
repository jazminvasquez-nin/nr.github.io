<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Nutria NR: ¡A comer manzanas!</title>
  <style>
    :root {
      --bg: #f5f7fb;
      --brand: #2d7a6b;
      --accent: #e74c3c;
      --ink: #1b1f23;
    }
    * { box-sizing: border-box; }
    html, body { height: 100%; }
    body {
      margin: 0;
      background: var(--bg);
      color: var(--ink);
      font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji", "Segoe UI Emoji";
      display: grid;
      place-items: center;
    }
    .wrap {
      width: min(100vw, 900px);
      height: min(100vh, 700px);
      display: grid;
      grid-template-rows: auto 1fr auto;
      gap: .5rem;
      padding: 1rem;
    }
    header {
      display: flex; align-items: center; justify-content: space-between;
      padding: .5rem 1rem; border-radius: 1rem;
      background: white; box-shadow: 0 6px 20px rgba(0,0,0,.06);
    }
    header .brand {
      display: inline-flex; gap: .6rem; align-items: center; font-weight: 700;
      color: var(--brand);
    }
    header .brand .dot { width: .7rem; height: .7rem; background: var(--brand); border-radius: 50%; }
    header .hud { display: flex; gap: 1rem; font-weight: 700; }
    header .badge { background: #eff7f5; color: var(--brand); padding: .35rem .6rem; border-radius: .6rem; }

    #gameArea {
      position: relative;
      border-radius: 1.25rem;
      overflow: hidden;
      background: linear-gradient(#dff7ff, #eef7ff 40%, #fefefe);
      box-shadow: 0 10px 30px rgba(0,0,0,.08);
    }
    canvas { display: block; width: 100%; height: 100%; }

    /* Overlays */
    .overlay {
      position: absolute; inset: 0; display: grid; place-items: center;
      background: rgba(255,255,255,.85);
      backdrop-filter: blur(3px);
      z-index: 2; text-align: center; padding: 2rem;
    }
    .card { background: white; border-radius: 1.25rem; padding: 1.5rem; box-shadow: 0 15px 40px rgba(0,0,0,.12); max-width: 540px; }
    .title { font-size: clamp(1.2rem, 3.4vw, 2rem); margin: 0 0 .5rem; }
    .muted { color: #5e6a74; margin: 0 0 1rem; }
    .row { display: flex; gap: .6rem; align-items: center; justify-content: center; flex-wrap: wrap; }
    button {
      appearance: none; border: 0; cursor: pointer; font-weight: 700;
      background: var(--brand); color: white; padding: .8rem 1.1rem; border-radius: .9rem;
      box-shadow: 0 8px 20px rgba(45,122,107,.35); transition: transform .12s ease;
    }
    button:hover { transform: translateY(-1px); }
    button.secondary { background: #eef7f5; color: var(--brand); box-shadow: none; }

    /* Touch controls */
    .controls {
      display: grid; grid-template-columns: 1fr 1fr 1fr; gap: .75rem;
      margin-top: .5rem;
    }
    .btn {
      background: white; border: 2px solid #e9eef2; border-radius: 1rem; padding: .9rem 1rem; text-align: center; font-weight: 700; cursor: pointer; user-select: none;
      box-shadow: 0 6px 18px rgba(0,0,0,.05);
    }
    .btn:active { transform: translateY(1px); }
    footer { color: #6b7785; font-size: .9rem; text-align: center; }
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <div class="brand"><span class="dot"></span> Nutria NR</div>
      <div class="hud">
        <div class="badge">Puntos: <span id="score">0</span></div>
        <div class="badge">Tiempo: <span id="time">60</span>s</div>
      </div>
    </header>

    <div id="gameArea">
      <canvas id="game"></canvas>

      <!-- START overlay -->
      <div id="startOverlay" class="overlay">
        <div class="card">
          <h1 class="title">¡A comer manzanas! 🍎</h1>
          <p class="muted">Mueve a la nutria con <b>◀︎ ▶︎</b> o arrastrando en la pantalla. Atrapa manzanas para sumar puntos en 60 segundos.</p>
          <div class="row">
            <button id="startBtn">Comenzar</button>
            <button id="muteBtn" class="secondary">🔊 Sonido</button>
          </div>
          <div class="controls" aria-hidden="true">
            <div class="btn" data-dir="left">◀︎</div>
            <div class="btn" data-dir="pause">⏸︎</div>
            <div class="btn" data-dir="right">▶︎</div>
          </div>
        </div>
      </div>

      <!-- GAME OVER overlay -->
      <div id="overOverlay" class="overlay" style="display:none">
        <div class="card">
          <h2 class="title">¡Tiempo! ⏱️</h2>
          <p class="muted">Puntaje final: <b id="finalScore">0</b></p>
          <div class="row">
            <button id="againBtn">Jugar otra vez</button>
            <button id="shareBtn" class="secondary">Compartir</button>
          </div>
        </div>
      </div>
    </div>

    <footer>
      Consejo: intenta encadenar <b>varias manzanas seguidas</b> para activar un pequeño combo 😉
    </footer>
  </div>

  <script>
    // ------- Configuración básica
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    const scoreEl = document.getElementById('score');
    const timeEl = document.getElementById('time');
    const startOverlay = document.getElementById('startOverlay');
    const overOverlay = document.getElementById('overOverlay');
    const finalScoreEl = document.getElementById('finalScore');

    const startBtn = document.getElementById('startBtn');
    const againBtn = document.getElementById('againBtn');
    const shareBtn = document.getElementById('shareBtn');
    const muteBtn = document.getElementById('muteBtn');

    // Carga de la imagen del personaje
    const playerImg = new Image();
    // Ubica el archivo "nutriamano.png" en la misma carpeta que este HTML
    playerImg.src = 'nutriamano.png';

    // Soniditos simples (WebAudio)
    const audio = {
      ctx: null,
      gain: null,
      enabled: true,
      init(){
        if(this.ctx) return;
        this.ctx = new (window.AudioContext || window.webkitAudioContext)();
        this.gain = this.ctx.createGain();
        this.gain.gain.value = 0.15; // volumen discreto
        this.gain.connect(this.ctx.destination);
      },
      beep(type='bonus'){
        if(!this.enabled) return;
        this.init();
        const o = this.ctx.createOscillator();
        const g = this.ctx.createGain();
        o.connect(g); g.connect(this.gain);
        const now = this.ctx.currentTime;
        const map = {
          bonus: [660, 880],
          tick: [330, 220],
          over: [180, 90]
        };
        const [f1,f2] = map[type] || map.bonus;
        o.frequency.setValueAtTime(f1, now);
        o.frequency.linearRampToValueAtTime(f2, now + 0.12);
        g.gain.setValueAtTime(0.0001, now);
        g.gain.exponentialRampToValueAtTime(1, now + 0.01);
        g.gain.exponentialRampToValueAtTime(0.0001, now + 0.18);
        o.start(now); o.stop(now + 0.2);
      }
    };

    muteBtn.addEventListener('click', () => {
      audio.enabled = !audio.enabled;
      muteBtn.textContent = audio.enabled ? '🔊 Sonido' : '🔇 Silencio';
    });

    // Mundo del juego
    let W = 800, H = 520; // resolución virtual
    function resize(){
      const rect = document.getElementById('gameArea').getBoundingClientRect();
      canvas.width = Math.floor(rect.width * window.devicePixelRatio);
      canvas.height = Math.floor(rect.height * window.devicePixelRatio);
      ctx.setTransform(window.devicePixelRatio,0,0,window.devicePixelRatio,0,0);
      W = rect.width; H = rect.height;
    }
    window.addEventListener('resize', resize, { passive:true });
    resize();

    const state = {
      running: false,
      timeLeft: 60,
      score: 0,
      combo: 0,
      lastCatchAt: 0
    };

    const player = {
      x: W/2, y: H-120, w: 110, h: 120, speed: 420, dir: 0,
      draw(){
        const scaleTo = 120; // altura aprox deseada
        const w = this.w, h = this.h;
        if(playerImg.complete && playerImg.naturalWidth){
          // Mantener relación de aspecto de la imagen original
          const ratio = playerImg.width / playerImg.height;
          const ph = Math.min(scaleTo, H*0.28);
          const pw = ph * ratio;
          this.w = pw; this.h = ph;
          ctx.drawImage(playerImg, this.x - pw/2, this.y - ph/2, pw, ph);
        } else {
          // Fallback (si no carga la imagen)
          ctx.fillStyle = '#c58a63';
          ctx.fillRect(this.x - w/2, this.y - h/2, w, h);
        }
      }
    };

    const keys = new Set();
    window.addEventListener('keydown', e => { if(['ArrowLeft','ArrowRight','a','d'].includes(e.key)) e.preventDefault(); keys.add(e.key); });
    window.addEventListener('keyup', e => keys.delete(e.key));

    // Controles táctiles
    const area = document.getElementById('gameArea');
    let touchX = null;
    area.addEventListener('touchstart', e => {
      if(!state.running) return;
      touchX = e.touches[0].clientX;
    }, {passive:true});
    area.addEventListener('touchmove', e => {
      if(!state.running || touchX==null) return;
      const x = e.touches[0].clientX;
      const delta = x - touchX; touchX = x;
      player.x += delta * 1.2; // sensibilidad
    }, {passive:true});
    area.addEventListener('touchend', ()=> touchX=null, {passive:true});

    // Botones táctiles del overlay (izq/pause/der)
    document.querySelectorAll('.btn').forEach(btn => {
      btn.addEventListener('click', () => {
        const d = btn.getAttribute('data-dir');
        if(d==='left') player.x -= 80;
        if(d==='right') player.x += 80;
        if(d==='pause') togglePause();
      });
    });

    const apples = [];
    function spawnApple(){
      const size = 26 + Math.random()*18;
      apples.push({ x: 40 + Math.random()*(W-80), y: -40, r: size/2, vy: 140 + Math.random()*120 });
    }

    let spawnTimer = 0;

    function drawApple(a){
      // Cuerpo
      ctx.fillStyle = '#d9392e';
      ctx.beginPath();
      ctx.arc(a.x, a.y, a.r, 0, Math.PI*2);
      ctx.fill();
      // Brillo
      ctx.globalAlpha = 0.2;
      ctx.fillStyle = '#fff';
      ctx.beginPath();
      ctx.arc(a.x - a.r*0.3, a.y - a.r*0.3, a.r*0.45, 0, Math.PI*2);
      ctx.fill();
      ctx.globalAlpha = 1;
      // Hoja
      ctx.fillStyle = '#2d7a6b';
      ctx.beginPath();
      ctx.ellipse(a.x + a.r*0.2, a.y - a.r*0.9, a.r*0.5, a.r*0.25, -0.5, 0, Math.PI*2);
      ctx.fill();
      // Rabito
      ctx.strokeStyle = '#5b3a29';
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.moveTo(a.x, a.y - a.r);
      ctx.quadraticCurveTo(a.x + 4, a.y - a.r*1.4, a.x + 10, a.y - a.r*1.2);
      ctx.stroke();
    }

    function rectsOverlap(ax, ay, aw, ah, bx, by, bw, bh){
      return Math.abs(ax - bx) * 2 < (aw + bw) && Math.abs(ay - by) * 2 < (ah + bh);
    }

    let lastTime = 0;
    function loop(ts){
      if(!state.running){ requestAnimationFrame(loop); return; }
      const dt = Math.min(0.033, (ts - lastTime)/1000 || 0); lastTime = ts;

      // Fondo (suelo suave)
      ctx.clearRect(0,0,W,H);
      const grd = ctx.createLinearGradient(0,0,0,H);
      grd.addColorStop(0,'#dff7ff'); grd.addColorStop(.5,'#eef7ff'); grd.addColorStop(1,'#ffffff');
      ctx.fillStyle = grd; ctx.fillRect(0,0,W,H);
      ctx.fillStyle = '#e9f3ef'; ctx.fillRect(0, H-40, W, 40);

      // Input teclado
      const speed = player.speed;
      if(keys.has('ArrowLeft') || keys.has('a')) player.x -= speed * dt;
      if(keys.has('ArrowRight') || keys.has('d')) player.x += speed * dt;

      // Limites
      player.x = Math.max(player.w/2, Math.min(W - player.w/2, player.x));

      // Spawns
      spawnTimer -= dt;
      if(spawnTimer <= 0){
        spawnApple();
        const base = 0.9; // tiempo entre spawns
        const difficulty = Math.max(0.35, base - (state.timeLeft/60)*0.5); // se acelera
        spawnTimer = difficulty;
      }

      // Mover manzanas
      for(let i=apples.length-1;i>=0;i--){
        const a = apples[i];
        a.y += a.vy * dt * (1 + (60-state.timeLeft)/80); // un poquito más rápido al final
        drawApple(a);
        // Colisión
        if(rectsOverlap(player.x, player.y, player.w*0.6, player.h*0.6, a.x, a.y, a.r*2, a.r*2)){
          apples.splice(i,1);
          const now = performance.now();
          state.combo = (now - state.lastCatchAt < 900) ? state.combo + 1 : 1;
          state.lastCatchAt = now;
          const bonus = Math.min(10, state.combo);
          state.score += 1 + Math.floor(bonus/3);
          scoreEl.textContent = state.score;
          audio.beep('bonus');
        }
        // Fuera de pantalla
        if(a.y - a.r > H){ apples.splice(i,1); }
      }

      // Jugador
      player.draw();

      // Timer
      timeEl.textContent = Math.ceil(state.timeLeft);
      state.timeLeft -= dt;
      if(state.timeLeft <= 0){
        gameOver();
      }

      requestAnimationFrame(loop);
    }

    function startGame(){
      state.running = true;
      state.timeLeft = 60; state.score = 0; state.combo = 0; state.lastCatchAt = 0;
      apples.length = 0; spawnTimer = 0; lastTime = 0;
      scoreEl.textContent = '0';
      startOverlay.style.display = 'none';
      overOverlay.style.display = 'none';
      audio.beep('tick');
    }

    function gameOver(){
      state.running = false;
      finalScoreEl.textContent = state.score;
      overOverlay.style.display = 'grid';
      audio.beep('over');
    }

    function togglePause(){
      state.running = !state.running;
      startOverlay.style.display = state.running ? 'none' : 'grid';
      if(state.running) requestAnimationFrame(loop);
    }

    // Botones
    startBtn.addEventListener('click', ()=>{ startGame(); requestAnimationFrame(loop); });
    againBtn.addEventListener('click', ()=>{ startGame(); });
    shareBtn.addEventListener('click', ()=>{
      const txt = `Acabo de hacer ${state.score} puntos con la Nutria NR 🍎`;
      if(navigator.share){ navigator.share({ title: 'Nutria NR', text: txt, url: location.href }).catch(()=>{}); }
      else { navigator.clipboard.writeText(txt).then(()=> alert('¡Puntaje copiado! Comparte tu marca 😉')); }
    });

    // Iniciar render para ver la pantalla inicial
    requestAnimationFrame(loop);
  </script>
</body>
</html>
