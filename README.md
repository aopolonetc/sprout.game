<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Спраут собирает цветочки</title>
<style>
  * { box-sizing: border-box; }
  body {
    margin: 0;
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    font-family: "Trebuchet MS", Arial, sans-serif;
    background: linear-gradient(160deg, #8fd3f4, #6fcf97);
    color: #123;
    padding: 12px;
    user-select: none;
    -webkit-user-select: none;
  }
  h1 {
    margin: 0 0 8px;
    font-size: 32px;
    color: #fff;
    text-shadow: 3px 3px 0 #00000044;
    text-align: center;
  }
  #hud {
    display: flex;
    gap: 18px;
    font-size: 24px;
    font-weight: bold;
    margin-bottom: 8px;
    background: #ffffffb0;
    padding: 8px 18px;
    border-radius: 14px;
  }
  #wrap {
    position: relative;
    width: 640px;
    max-width: 100%;
  }
  canvas {
    display: block;
    width: 100%;
    height: auto;
    background: #bff0a0;
    border: 8px solid #ffd93d;
    border-radius: 18px;
  }
  .screen {
    position: absolute;
    inset: 0;
    background: #0b3d2ee6;
    border-radius: 18px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    padding: 20px;
    gap: 14px;
    color: #fff;
    z-index: 10;
  }
  .screen h2 { font-size: 32px; margin: 0; }
  .screen p { font-size: 20px; margin: 0; line-height: 1.4; max-width: 460px; }
  
  .menu-btn {
    font-family: inherit;
    font-size: 26px;
    font-weight: bold;
    padding: 14px 36px;
    border: none;
    border-radius: 16px;
    background: #ffd93d;
    color: #16432a;
    cursor: pointer;
    box-shadow: 0 6px 0 #c9a400;
  }
  .menu-btn:active { transform: translateY(4px); box-shadow: 0 2px 0 #c9a400; }
  
  .hidden { display: none !important; }
  .hint { font-size: 18px; opacity: .95; margin-top: 6px; }

  #controls {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    margin-top: 14px;
    width: 100%;
    max-width: 320px;
  }
  .ctrl-row {
    display: flex;
    gap: 14px;
    justify-content: center;
    width: 100%;
  }
  .ctrl-btn {
    width: 64px;
    height: 64px;
    font-size: 32px;
    font-weight: bold;
    background: #ffd93d;
    color: #16432a;
    border: 4px solid #fff;
    border-radius: 20px;
    box-shadow: 0 6px 0 #c9a400;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    touch-action: none;
    user-select: none;
    -webkit-user-select: none;
  }
  .ctrl-btn:active {
    transform: translateY(4px);
    box-shadow: 0 2px 0 #c9a400;
    background: #ffea75;
  }
</style>
</head>
<body>

<h1>🌼 Спраут собирает цветочки на поляне</h1>
<div id="hud">
  <span>Цветочки: <span id="score">0</span>/8</span>
  <span>Время: <span id="time">45</span></span>
</div>

<div id="wrap">
  <canvas id="game" width="640" height="440"></canvas>

  <div id="startScreen" class="screen">
    <h2>Привет, я Спраут!</h2>
    <p>Собери все <b>8 цветочков</b> на поляне за 45 секунд.<br>Берегись <b>птиц</b> — они гоняются за тобой!</p>
    <button id="startBtn" class="menu-btn">Старт</button>
    <p class="hint">Управление: стрелки, WASD или кнопки внизу</p>
  </div>

  <div id="endScreen" class="screen hidden">
    <h2 id="endTitle">Победа!</h2>
    <p id="endText"></p>
    <button id="againBtn" class="menu-btn">Играть ещё раз</button>
  </div>
</div>

<div id="controls">
  <div class="ctrl-row">
    <button class="ctrl-btn" id="btnUp">▲</button>
  </div>
  <div class="ctrl-row">
    <button class="ctrl-btn" id="btnLeft">◀</button>
    <button class="ctrl-btn" id="btnDown">▼</button>
    <button class="ctrl-btn" id="btnRight">▶</button>
  </div>
</div>

<script>
const cv = document.getElementById('game');
const ctx = cv.getContext('2d');
const W = cv.width, H = cv.height;

const scoreEl = document.getElementById('score');
const timeEl  = document.getElementById('time');
const startScreen = document.getElementById('startScreen');
const endScreen   = document.getElementById('endScreen');
const endTitle = document.getElementById('endTitle');
const endText  = document.getElementById('endText');

const GOAL = 8;
const TIME_LIMIT = 45;

let running = false;
let score = 0;
let timeLeft = TIME_LIMIT;
let timerId = null;

const hero = { x: W / 2, y: H / 2, r: 24, speed: 4.5, face: 1 };
let flowers = [];
let birds = [];
const keys = {};

document.addEventListener('keydown', e => {
  const k = e.key.toLowerCase();
  keys[k] = true;
  if (['arrowup','arrowdown','arrowleft','arrowright',' '].includes(e.key)) e.preventDefault();
});
document.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

function setupTouchButton(id, keyName) {
  const btn = document.getElementById(id);
  
  const press = (e) => {
    e.preventDefault();
    keys[keyName] = true;
  };
  
  const release = (e) => {
    e.preventDefault();
    keys[keyName] = false;
  };

  btn.addEventListener('mousedown', press);
  btn.addEventListener('mouseup', release);
  btn.addEventListener('mouseleave', release);
  
  btn.addEventListener('touchstart', press, { passive: false });
  btn.addEventListener('touchend', release, { passive: false });
  btn.addEventListener('touchcancel', release, { passive: false });
}

setupTouchButton('btnUp', 'arrowup');
setupTouchButton('btnDown', 'arrowdown');
setupTouchButton('btnLeft', 'arrowleft');
setupTouchButton('btnRight', 'arrowright');

function rand(a, b) { return a + Math.random() * (b - a); }
function dist(dx, dy) { return Math.sqrt(dx * dx + dy * dy); }

function farFromHero(x, y, minDist) {
  return dist(x - hero.x, y - hero.y) >= minDist;
}

function placeFlowers() {
  flowers = [];
  let tries = 0;
  while (flowers.length < GOAL && tries < 500) {
    tries++;
    const x = rand(30, W - 30);
    const y = rand(30, H - 30);
    if (!farFromHero(x, y, 90)) continue;
    let ok = true;
    for (const f of flowers) {
      if (dist(x - f.x, y - f.y) < 55) { ok = false; break; }
    }
    if (ok) flowers.push({ x, y, r: 14, taken: false, sway: rand(0, Math.PI * 2) });
  }
}

function makeBird() {
  let x, y;
  do {
    x = rand(30, W - 30);
    y = rand(30, H - 30);
  } while (!farFromHero(x, y, 160));
  const angle = rand(0, Math.PI * 2);
  const speed = rand(1.6, 2.4);
  birds.push({ x, y, r: 16, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed, wing: 0 });
}

function reset() {
  score = 0;
  timeLeft = TIME_LIMIT;
  hero.x = W / 2; hero.y = H / 2; hero.face = 1;
  birds = [];
  placeFlowers();
  for (let i = 0; i < 3; i++) makeBird();
  scoreEl.textContent = score;
  timeEl.textContent = timeLeft;
}

function startGame() {
  reset();
  startScreen.classList.add('hidden');
  endScreen.classList.add('hidden');
  running = true;
  clearInterval(timerId);
  timerId = setInterval(() => {
    if (!running) return;
    timeLeft--;
    timeEl.textContent = timeLeft;
    if (timeLeft <= 0) gameOver(false, 'Время вышло! Собрано цветочков: ' + score + ' из ' + GOAL);
  }, 1000);
}

function gameOver(win, text) {
  running = false;
  clearInterval(timerId);
  endTitle.textContent = win ? '🏆 Победа!' : '💥 Проигрыш';
  endText.textContent = text;
  endScreen.classList.remove('hidden');
}

document.getElementById('startBtn').onclick = startGame;
document.getElementById('againBtn').onclick = startGame;

function update() {
  if (!running) return;

  let dx = 0, dy = 0;
  if (keys['arrowleft'] || keys['a']) dx -= 1;
  if (keys['arrowright'] || keys['d']) dx += 1;
  if (keys['arrowup'] || keys['w']) dy -= 1;
  if (keys['arrowdown'] || keys['s']) dy += 1;
  
  if (dx || dy) {
    const len = dist(dx, dy);
    hero.x += dx / len * hero.speed;
    hero.y += dy / len * hero.speed;
    if (dx !== 0) hero.face = dx > 0 ? 1 : -1;
  }
  hero.x = Math.max(hero.r, Math.min(W - hero.r, hero.x));
  hero.y = Math.max(hero.r, Math.min(H - hero.r, hero.y));

  for (const b of birds) {
    const toX = hero.x - b.x, toY = hero.y - b.y;
    const d = dist(toX, toY) || 1;
    const chase = 0.03;
    b.vx += (toX / d) * chase;
    b.vy += (toY / d) * chase;
    const spd = dist(b.vx, b.vy);
    const maxSpd = 2.6;
    if (spd > maxSpd) { b.vx = b.vx / spd * maxSpd; b.vy = b.vy / spd * maxSpd; }
    b.x += b.vx; b.y += b.vy;
    b.wing += 0.3;
    if (b.x < b.r || b.x > W - b.r) { b.vx *= -1; b.x = Math.max(b.r, Math.min(W - b.r, b.x)); }
    if (b.y < b.r || b.y > H - b.r) { b.vy *= -1; b.y = Math.max(b.r, Math.min(H - b.r, b.y)); }
    if (dist(b.x - hero.x, b.y - hero.y) < b.r + hero.r - 8) {
      gameOver(false, 'Птица поймала Спраута! Собрано цветочков: ' + score + ' из ' + GOAL);
      return;
    }
  }

  for (const f of flowers) {
    if (f.taken) continue;
    if (dist(f.x - hero.x, f.y - hero.y) < f.r + hero.r) {
      f.taken = true;
      score++;
      scoreEl.textContent = score;
      if (score % 3 === 0 && birds.length < 6) makeBird();
      if (score >= GOAL) { gameOver(true, 'Все цветочки собраны! Осталось секунд: ' + timeLeft); return; }
    }
  }
}

function drawFlower(f) {
  if (f.taken) return;
  const sway = Math.sin(Date.now() / 400 + f.sway) * 3;
  ctx.save();
  ctx.translate(f.x + sway, f.y);
  ctx.strokeStyle = '#2e7d32';
  ctx.lineWidth = 4;
  ctx.beginPath(); ctx.moveTo(0, f.r + 10); ctx.lineTo(0, 4); ctx.stroke();
  const colors = ['#ff6b81', '#ffd93d', '#ff9f43', '#eb3b5a', '#a55eea', '#fd79a8'];
  const col = colors[Math.floor((f.sway * 10) % colors.length)];
  ctx.fillStyle = col;
  for (let i = 0; i < 5; i++) {
    const a = (Math.PI * 2 / 5) * i;
    ctx.beginPath();
    ctx.ellipse(Math.cos(a) * 9, Math.sin(a) * 9, 7, 4, a, 0, Math.PI * 2);
    ctx.fill();
  }
  ctx.fillStyle = '#ffe066';
  ctx.beginPath(); ctx.arc(0, 0, 6, 0, Math.PI * 2); ctx.fill();
  ctx.restore();
}

function drawBird(b) {
  ctx.save();
  ctx.translate(b.x, b.y);
  const flap = Math.sin(b.wing) * 10;
  ctx.fillStyle = '#37474f';
  ctx.beginPath(); ctx.ellipse(0, 0, 12, 8, 0, 0, Math.PI * 2); ctx.fill();
  ctx.beginPath();
  ctx.moveTo(-4, 0); ctx.lineTo(-20, -flap); ctx.lineTo(-4, 4); ctx.closePath(); ctx.fill();
  ctx.beginPath();
  ctx.moveTo(4, 0); ctx.lineTo(20, -flap); ctx.lineTo(4, 4); ctx.closePath(); ctx.fill();
  ctx.fillStyle = '#ffb142';
  ctx.beginPath(); ctx.moveTo(10, -2); ctx.lineTo(18, 0); ctx.lineTo(10, 2); ctx.closePath(); ctx.fill();
  ctx.fillStyle = '#fff';
  ctx.beginPath(); ctx.arc(6, -4, 3, 0, Math.PI * 2); ctx.fill();
  ctx.fillStyle = '#000';
  ctx.beginPath(); ctx.arc(7, -4, 1.5, 0, Math.PI * 2); ctx.fill();
  ctx.restore();
}

function drawHero() {
  const t = Date.now() / 300;
  ctx.save();
  ctx.translate(hero.x, hero.y);
  ctx.scale(hero.face, 1);

  ctx.fillStyle = 'rgba(0,0,0,0.15)';
  ctx.beginPath();
  ctx.ellipse(0, 32, 24, 7, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#1e1c52';
  ctx.strokeStyle = '#0e0b30';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.roundRect(-16, 20, 10, 14, 3);
  ctx.roundRect(6, 20, 10, 14, 3);
  ctx.fill(); ctx.stroke();

  ctx.strokeStyle = '#5c6bc0';
  ctx.lineWidth = 7;
  ctx.lineCap = 'round';
  ctx.beginPath();
  ctx.moveTo(-18, 10);
  ctx.quadraticCurveTo(-28, 12, -26 + Math.sin(t)*2, 22);
  ctx.stroke();
  ctx.beginPath();
  ctx.moveTo(18, 10);
  ctx.quadraticCurveTo(28, 12, 26 - Math.sin(t)*2, 22);
  ctx.stroke();

  ctx.fillStyle = '#ff4081';
  ctx.strokeStyle = '#c2185b';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.arc(-26 + Math.sin(t)*2, 22, 6, 0, Math.PI*2);
  ctx.fill(); ctx.stroke();
  ctx.beginPath();
  ctx.arc(26 - Math.sin(t)*2, 22, 6, 0, Math.PI*2);
  ctx.fill(); ctx.stroke();

  ctx.fillStyle = '#f5f5f5';
  ctx.strokeStyle = '#b0bec5';
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.roundRect(-22, 2, 44, 22, [4, 4, 12, 12]);
  ctx.fill(); ctx.stroke();

  ctx.fillStyle = '#00e5ff';
  ctx.strokeStyle = '#0288d1';
  ctx.lineWidth = 1.5;
  ctx.beginPath();
  ctx.arc(0, 13, 4, 0, Math.PI*2);
  ctx.fill(); ctx.stroke();
  
  ctx.fillStyle = '#ff1744';
  ctx.beginPath();
  ctx.arc(11, 8, 2.5, 0, Math.PI*2);
  ctx.fill();

  ctx.fillStyle = '#f48fb1';
  ctx.beginPath();
  ctx.arc(0, -5, 17, 0, Math.PI*2);
  ctx.fill();
  
  ctx.fillStyle = '#ffffff';
  ctx.beginPath();
  ctx.arc(-11, 1, 4, 0, Math.PI*2);
  ctx.arc(11, 1, 4, 0, Math.PI*2);
  ctx.fill();

  ctx.save();
  ctx.rotate(0.04);
  ctx.fillStyle = '#ffd54f';
  ctx.strokeStyle = '#ffb300';
  ctx.lineWidth = 1.5;
  ctx.beginPath();
  ctx.roundRect(-11, -14, 22, 18, 2);
  ctx.fill(); ctx.stroke();

  ctx.fillStyle = '#eceff1';
  ctx.strokeStyle = '#b0bec5';
  ctx.lineWidth = 1;
  ctx.beginPath();
  ctx.moveTo(-4, -16);
  ctx.lineTo(4, -16);
  ctx.lineTo(2, -12);
  ctx.lineTo(-2, -12);
  ctx.closePath();
  ctx.fill(); ctx.stroke();

  ctx.fillStyle = '#212121';
  ctx.beginPath();
  ctx.ellipse(-5, -6, 2.2, 4.5, 0, 0, Math.PI*2);
  ctx.fill();
  ctx.beginPath();
  ctx.ellipse(5, -6, 2.2, 4.5, 0, 0, Math.PI*2);
  ctx.fill();
  ctx.strokeStyle = '#212121';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(-2, -1);
  ctx.lineTo(2, -1);
  ctx.stroke();
  ctx.restore();

  ctx.fillStyle = 'rgba(129, 212, 250, 0.3)';
  ctx.strokeStyle = '#81d4fa';
  ctx.lineWidth = 2.5;
  ctx.beginPath();
  ctx.arc(0, -6, 21, Math.PI, 0);
  ctx.lineTo(21, 3);
  ctx.lineTo(-21, 3);
  ctx.closePath();
  ctx.fill(); ctx.stroke();

  ctx.fillStyle = 'rgba(255, 255, 255, 0.45)';
  ctx.beginPath();
  ctx.ellipse(-10, -18, 6, 2.5, -Math.PI/6, 0, Math.PI*2);
  ctx.fill();

  ctx.strokeStyle = '#81c784';
  ctx.lineWidth = 3.5;
  ctx.beginPath();
  ctx.moveTo(0, -26);
  ctx.lineTo(0, -32);
  ctx.stroke();

  ctx.fillStyle = '#00c853';
  ctx.strokeStyle = '#1b5e20';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.ellipse(0, -38, 14, 8, -0.05, 0, Math.PI*2);
  ctx.fill(); ctx.stroke();

  ctx.restore();
}

function draw() {
  ctx.fillStyle = '#bff0a0';
  ctx.fillRect(0, 0, W, H);
  ctx.strokeStyle = '#a6dd8a';
  ctx.lineWidth = 2;
  for (let x = 0; x < W; x += 40) {
    ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke();
  }
  for (let y = 0; y < H; y += 40) {
    ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke();
  }

  flowers.forEach(drawFlower);
  birds.forEach(drawBird);
  drawHero();
}

function loop() {
  update();
  draw();
  requestAnimationFrame(loop);
}

if (!CanvasRenderingContext2D.prototype.roundRect) {
  CanvasRenderingContext2D.prototype.roundRect = function (x, y, w, h, r) {
    this.beginPath();
    this.moveTo(x + r, y);
    this.arcTo(x + w, y, x + w, y + h, r);
    this.arcTo(x + w, y + h, x, y + h, r);
    this.arcTo(x, y + h, x, y, r);
    this.arcTo(x, y, x + w, y, r);
    this.closePath();
    return this;
  };
}

reset();
running = false;
loop();
</script>
</body>
</html>

