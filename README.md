<!doctype html>
<html lang="de">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Jump'n'Run — Demo</title>
<style>
  :root{
    --bg:#9ad0ff;
    --ground:#5a3c1e;
    --platform:#8b5e3c;
    --player:#ff595e;
    --enemy:#22333b;
    --coin:#ffd166;
    --hud:#023047;
    font-family: Inter, ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
  }
  html,body{height:100%;margin:0;background:linear-gradient(#bfe9ff, var(--bg));}
  .wrap{display:flex;flex-direction:column;height:100vh;align-items:center;justify-content:center;padding:1rem;box-sizing:border-box;}
  h1{margin:0 0 .5rem;font-size:1.25rem;color:var(--hud);}
  #gameContainer{width:100%;max-width:960px;background:linear-gradient(#ffffff33, #00000005);border-radius:12px;box-shadow:0 10px 30px rgba(2,48,71,0.12);overflow:hidden;display:flex;flex-direction:column;}
  canvas{display:block;width:100%;height:auto;background:linear-gradient(#a7e0ff, #87c9ff);image-rendering:pixelated;}
  .hud{display:flex;justify-content:space-between;padding:.5rem 1rem;background:transparent;color:var(--hud);align-items:center;font-weight:600;}
  .controls{font-size:.86rem;opacity:0.9;}
  .footer{display:flex;gap:.5rem;padding:.5rem 1rem;align-items:center;border-top:1px solid #00000010;background:#ffffff11;}
  button{padding:.4rem .6rem;border-radius:8px;border:0;background:var(--hud);color:white;cursor:pointer;font-weight:600}
  button:active{transform:translateY(1px)}
  /* mobile touch buttons (only visible on small screens) */
  .touch-controls{display:none;position:absolute;left:0;right:0;bottom:18px;pointer-events:none;justify-content:space-between;padding:0 20px;}
  .touch-controls .btn{pointer-events:auto;background:#ffffff88;border-radius:999px;padding:12px 18px;font-weight:700;backdrop-filter: blur(4px);}
  @media (max-width:640px){
    .touch-controls{display:flex;}
  }
</style>
</head>
<body>
<div class="wrap">
  <h1>Jump'n'Run — Demo</h1>
  <div id="gameContainer">
    <div class="hud">
      <div>Score: <span id="score">0</span></div>
      <div class="controls">Steuerung: ← → = laufen, ↑ / Leertaste = springen</div>
      <div id="status">Leben: <span id="lives">3</span></div>
    </div>
    <canvas id="game"></canvas>
    <div class="footer">
      <div style="flex:1">Ziel: Sammle Münzen (+10), vermeide Gegner oder spring auf sie (+20).</div>
      <div><button id="restart">Neu starten</button></div>
    </div>
    <!-- mobile buttons -->
    <div class="touch-controls" aria-hidden="true">
      <div style="display:flex;gap:8px;">
        <div class="btn" id="leftBtn">◀</div>
        <div class="btn" id="rightBtn">▶</div>
      </div>
      <div class="btn" id="jumpBtn">▲</div>
    </div>
  </div>
</div>

<script>
/*
  Einfaches Jump'n'Run
  - Canvas-basiert
  - Physik: Gravitation, Bodenprüfung, AABB-Kollision
  - Level: Plattformen, Münzen, Gegner
  - Steuerung: Tastatur + Touch (mobil)
  - Einzel-Datei; leicht erweiterbar
*/

/* ---- Setup ---- */
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d', { alpha: false });
let DPR = Math.max(1, window.devicePixelRatio || 1);

// resize canvas to container width keeping a 16:9-ish gameplay area
function resize() {
  const container = document.getElementById('gameContainer');
  const styleWidth = container.clientWidth;
  // game logical width/height
  const logicalW = 960; // logical coordinate width
  const scale = styleWidth / logicalW;
  const logicalH = Math.round(540); // 16:9
  canvas.style.width = styleWidth + 'px';
  canvas.style.height = Math.round(logicalH * scale) + 'px';
  canvas.width = Math.round(styleWidth * DPR);
  canvas.height = Math.round((logicalH * scale) * DPR);
  ctx.setTransform(DPR,0,0,DPR,0,0);
}
window.addEventListener('resize', () => { DPR = Math.max(1, window.devicePixelRatio || 1); resize(); });
resize();

/* ---- Game variables ---- */
let lastTime = 0;
let keys = {};
let scoreEl = document.getElementById('score');
let livesEl = document.getElementById('lives');
let restartBtn = document.getElementById('restart');

const GAME = {
  gravity: 1600,            // px/s^2
  friction: 0.85,
  playerSpeed: 380,        // horizontal speed px/s
  jumpVel: 620,            // initial jump velocity px/s
  cameraX: 0,
  width: 960,
  height: 540
};

/* ---- Entities ---- */
class Rect {
  constructor(x,y,w,h){this.x=x;this.y=y;this.w=w;this.h=h;}
  get left(){return this.x}
  get right(){return this.x+this.w}
  get top(){return this.y}
  get bottom(){return this.y+this.h}
  intersects(other){
    return !(this.right<=other.left || this.left>=other.right || this.bottom<=other.top || this.top>=other.bottom);
  }
}

class Player extends Rect {
  constructor(x,y){
    super(x,y,36,48);
    this.vx = 0;
    this.vy = 0;
    this.canJump = false;
    this.onGround = false;
    this.color = getComputedStyle(document.documentElement).getPropertyValue('--player').trim() || '#ff595e';
    this.lives = 3;
  }
  update(dt){
    // horizontal input
    let inputDir = 0;
    if(keys.left) inputDir -= 1;
    if(keys.right) inputDir += 1;
    // apply horizontal speed (instant acceleration for simplicity)
    this.vx = inputDir * GAME.playerSpeed;
    // apply gravity
    this.vy += GAME.gravity * dt;
    // apply velocity
    this.x += this.vx * dt;
    this.y += this.vy * dt;
    // simple world bounds
    if(this.y > 3000){ this.respawn(); loseLife(); }
  }
  respawn(){
    this.x = 120; this.y = 50; this.vx=0; this.vy=0;
  }
}

/* ---- Level ---- */
let player;
let platforms = [];
let coins = [];
let enemies = [];
let score = 0;
let gameOver = false;

function makeLevel(){
  // reset
  platforms = [];
  coins = [];
  enemies = [];
  score = 0;
  gameOver = false;

  // ground (long)
  platforms.push(new Rect(-1000, 480, 4000, 120));
  // some floating platforms
  platforms.push(new Rect(80, 380, 160, 20));
  platforms.push(new Rect(300, 300, 140, 20));
  platforms.push(new Rect(540, 240, 160, 20));
  platforms.push(new Rect(760, 320, 120, 20));
  platforms.push(new Rect(980, 260, 160, 20));
  platforms.push(new Rect(1220, 340, 220, 20));
  platforms.push(new Rect(1520, 280, 140, 20));
  platforms.push(new Rect(1760, 360, 280, 20));
  platforms.push(new Rect(2100, 300, 220, 20));
  // coins placed above some platforms
  [{x:120,y:330},{x:340,y:250},{x:600,y:190},{x:820,y:270},{x:1000,y:210},
   {x:1260,y:290},{x:1540,y:230},{x:1880,y:310},{x:2140,y:250}].forEach(c=>coins.push({x:c.x,y:c.y,r:10,collected:false}));
  // enemies patrol on some platforms
  enemies.push({x:360,y:268,w:34,h:34,vx:80,dir:1,range:[300,420]});
  enemies.push({x:1700,y:328,w:36,h:36,vx:60,dir:1,range:[1600,1980]});
  // player
  player = new Player(120,50);
  player.lives = 3;
  updateHUD();
}

/* ---- Collision helpers ---- */
function resolveCollisions(rect){
  rect.onGround = false;
  // check each platform, do AABB resolution (only vertical resolution for simpler platformer)
  for(const p of platforms){
    if(rect.intersects(p)){
      // we will push object outside along the smallest penetration axis
      const overlapX = Math.min(rect.right - p.left, p.right - rect.left);
      const overlapY = Math.min(rect.bottom - p.top, p.bottom - rect.top);
      if(overlapY < overlapX){
        // vertical collision
        if(rect.vy > 0 && rect.bottom - p.top <= overlapY + 5){
          // landed on top of platform
          rect.y = p.top - rect.h;
          rect.vy = 0;
          rect.onGround = true;
          rect.canJump = true;
        } else {
          // hit head on bottom of platform
          rect.y = p.bottom;
          rect.vy = Math.max(0, rect.vy);
        }
      } else {
        // horizontal collision - push out
        if(rect.x < p.x) rect.x = p.left - rect.w;
        else rect.x = p.right;
        rect.vx = 0;
      }
    }
  }
}

/* ---- Game logic ---- */
function update(dt){
  if(gameOver) return;
  // player update
  player.update(dt);
  resolveCollisions(player);

  // camera follows player (simple)
  GAME.cameraX = clamp(player.x - 240, 0, 2400);

  // coins collection
  for(const coin of coins){
    if(!coin.collected){
      const coinRect = new Rect(coin.x-coin.r, coin.y-coin.r, coin.r*2, coin.r*2);
      if(player.intersects(coinRect)){
        coin.collected = true;
        score += 10;
        updateHUD();
      }
    }
  }

  // enemies update + collisions
  for(const en of enemies){
    en.x += en.vx * en.dir * dt;
    if(en.x < en.range[0]) en.dir = 1;
    if(en.x > en.range[1]) en.dir = -1;
    // simple AABB check
    const enemyRect = new Rect(en.x, en.y, en.w, en.h);
    if(player.intersects(enemyRect)){
      // if player is falling and hits top of enemy -> "stomp" enemy
      if(player.vy > 80 && (player.bottom - enemyRect.top) < 30){
        // stomp
        score += 20;
        // knock enemy away (remove)
        en.dead = true;
        player.vy = -260; // bounce
        updateHUD();
      } else {
        // player hit -> lose life and respawn
        player.respawn();
        loseLife();
      }
    }
  }
  // remove dead enemies
  enemies = enemies.filter(e => !e.dead);

  // simple win condition: collect all coins
  if(coins.every(c => c.collected)){
    gameOver = true;
    setTimeout(()=> alert('Du hast alle Münzen gesammelt! 🎉\nPunkte: ' + score), 120);
  }
}

/* ---- Rendering ---- */
function clear(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
}
function draw(){
  // logical size we render to (use CSS px coordinates)
  const W = canvas.clientWidth;
  const H = canvas.clientHeight;
  // background parallax
  ctx.save();
  ctx.fillStyle = '#7ad0ff';
  ctx.fillRect(0,0,W,H);

  // sky gradient (simple)
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0,'#bfe9ff');
  g.addColorStop(1,'#87c9ff');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,W,H);

  // translate by camera
  ctx.translate(-GAME.cameraX + 120, 0);

  // draw distant hills
  ctx.fillStyle = '#d0f3d6';
  ctx.beginPath();
  ctx.ellipse(500,430,420,120,0,0,Math.PI*2);
  ctx.ellipse(1400,430,520,150,0,0,Math.PI*2);
  ctx.fill();

  // draw platforms
  for(const p of platforms){
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--platform').trim() || '#8b5e3c';
    roundRect(ctx, p.x, p.y, p.w, p.h, 6);
    ctx.fill();
    // top highlight
    ctx.fillStyle = 'rgba(255,255,255,0.06)';
    ctx.fillRect(p.x, p.y, p.w, 6);
  }

  // coins
  for(const coin of coins){
    if(coin.collected) continue;
    ctx.beginPath();
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--coin').trim() || '#ffd166';
    ctx.arc(coin.x, coin.y, coin.r, 0, Math.PI*2);
    ctx.fill();
    ctx.fillStyle = 'rgba(255,255,255,0.2)';
    ctx.fillRect(coin.x-coin.r/2, coin.y-coin.r/2-2, coin.r, 2);
  }

  // enemies
  for(const en of enemies){
    ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--enemy').trim() || '#22333b';
    ctx.fillRect(en.x, en.y, en.w, en.h);
    // simple eye
    ctx.fillStyle = '#fff';
    ctx.fillRect(en.x + en.w/3 * (en.dir>0?1:0.2), en.y + en.h/4, 6, 6);
  }

  // player
  ctx.fillStyle = player.color;
  roundRect(ctx, player.x, player.y, player.w, player.h, 6);
  ctx.fill();

  // HUD drawn in DOM, so no need to draw text here
  ctx.restore();
}

function roundRect(ctx,x,y,w,h,r){
  ctx.beginPath();
  ctx.moveTo(x+r,y);
  ctx.arcTo(x+w,y,x+w,y+h,r);
  ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r);
  ctx.arcTo(x,y,x+w,y,r);
  ctx.closePath();
}

/* ---- Input ---- */
window.addEventListener('keydown', e=>{
  if(e.code === 'ArrowLeft' || e.code === 'KeyA') keys.left = true;
  if(e.code === 'ArrowRight' || e.code === 'KeyD') keys.right = true;
  if(e.code === 'ArrowUp' || e.code === 'Space' || e.code === 'KeyW'){ 
    if(player.canJump || player.onGround){ player.vy = -GAME.jumpVel; player.canJump = false; player.onGround = false; }
  }
  if(e.code === 'KeyR'){ makeLevel(); }
});
window.addEventListener('keyup', e=>{
  if(e.code === 'ArrowLeft' || e.code === 'KeyA') keys.left = false;
  if(e.code === 'ArrowRight' || e.code === 'KeyD') keys.right = false;
});

 // touch controls
const leftBtn = document.getElementById('leftBtn');
const rightBtn = document.getElementById('rightBtn');
const jumpBtn = document.getElementById('jumpBtn');
leftBtn.addEventListener('touchstart', (e)=>{ e.preventDefault(); keys.left = true; });
leftBtn.addEventListener('touchend', (e)=>{ e.preventDefault(); keys.left = false; });
rightBtn.addEventListener('touchstart', (e)=>{ e.preventDefault(); keys.right = true; });
rightBtn.addEventListener('touchend', (e)=>{ e.preventDefault(); keys.right = false; });
jumpBtn.addEventListener('touchstart', (e)=>{ e.preventDefault(); if(player.canJump || player.onGround){ player.vy = -GAME.jumpVel; player.canJump = false; player.onGround = false; } });

/* ---- HUD & helpers ---- */
function updateHUD(){ scoreEl.textContent = score; livesEl.textContent = player.lives; }
function loseLife(){
  player.lives -= 1;
  updateHUD();
  if(player.lives <= 0){
    gameOver = true;
    setTimeout(()=> {
      if(confirm('Game over. Noch einmal?')){
        makeLevel();
      }
    }, 50);
  }
}

/* ---- Game loop ---- */
function frame(ts){
  if(!lastTime) lastTime = ts;
  const dt = Math.min(0.033, (ts - lastTime) / 1000); // clamp dt to avoid big jumps
  lastTime = ts;
  update(dt);
  clear();
  draw();
  requestAnimationFrame(frame);
}

/* ---- Utility ---- */
function clamp(v,min,max){ return Math.max(min, Math.min(max, v)); }

/* ---- Start ---- */
restartBtn.addEventListener('click', ()=> makeLevel());
makeLevel();
requestAnimationFrame(frame);

</script>
</body>
</html>
