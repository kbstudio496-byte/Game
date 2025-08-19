<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Mini Battle Royale</title>
<style>
  html, body { margin:0; height:100%; background:#0f1215; font-family:system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; color:#e8eaed; }
  #ui { position:fixed; inset:0; pointer-events:none; }
  .hud {
    position: fixed; left: 12px; bottom: 12px; pointer-events: none;
    background: rgba(15,18,21,0.6); border: 1px solid rgba(255,255,255,0.07);
    padding: 10px 12px; border-radius: 10px; backdrop-filter: blur(6px);
    box-shadow: 0 8px 24px rgba(0,0,0,0.35);
  }
  .hud span { display:inline-block; min-width: 68px; }
  .topright {
    position:fixed; right:12px; top:12px; pointer-events:none;
    background: rgba(15,18,21,0.6); border: 1px solid rgba(255,255,255,0.07);
    padding: 10px 12px; border-radius: 10px; backdrop-filter: blur(6px);
    box-shadow: 0 8px 24px rgba(0,0,0,0.35);
  }
  .centerBanner {
    position:fixed; left:50%; top:10px; transform:translateX(-50%);
    background: rgba(15,18,21,0.6); border: 1px solid rgba(255,255,255,0.07);
    padding: 8px 12px; border-radius: 10px; backdrop-filter: blur(6px); pointer-events:none;
  }
  .bigmodal {
    position: fixed; inset:0; display:none; align-items:center; justify-content:center;
    background: rgba(0,0,0,0.6); backdrop-filter: blur(4px);
  }
  .card {
    background:#111418; border:1px solid rgba(255,255,255,0.08); border-radius:16px; padding:20px;
    width:min(560px, 92vw); color:#eef1f5; text-align:center; box-shadow: 0 20px 60px rgba(0,0,0,0.45);
  }
  .card h1 { margin: 0 0 6px 0; font-size: 26px; letter-spacing: 0.2px; }
  .card p { opacity:0.85; }
  .card button {
    pointer-events:auto; margin-top:14px; background:#2d7dff; color:#fff; border:0; padding:10px 16px;
    border-radius: 10px; cursor:pointer; font-weight:600;
  }
  .minimap {
    position:fixed; right:12px; bottom:12px; pointer-events:none;
    background: rgba(15,18,21,0.6); border: 1px solid rgba(255,255,255,0.07);
    width: 140px; height: 140px; border-radius: 10px; overflow:hidden;
  }
  canvas { display:block; width:100vw; height:100vh; }
</style>
</head>
<body>
<canvas id="game" width="1280" height="720"></canvas>

<div id="ui">
  <div class="centerBanner" id="phaseText">Prepare to drop…</div>
  <div class="hud" id="hud">
    <div><span>HP:</span><b id="hp">100</b></div>
    <div><span>Ammo:</span><b id="ammo">60</b></div>
    <div><span>Kills:</span><b id="kills">0</b></div>
  </div>
  <div class="topright">
    <div><b>Alive:</b> <span id="alive">-</span></div>
    <div><b>Phase:</b> <span id="phase">1</span></div>
    <div><b>Time:</b> <span id="time">0:00</span></div>
  </div>
  <canvas class="minimap" id="mini" width="140" height="140"></canvas>
</div>

<div class="bigmodal" id="modal">
  <div class="card">
    <h1 id="modalTitle">Winner Winner!</h1>
    <p id="modalSubtitle">You survived the island.</p>
    <button id="playBtn">Play Again</button>
    <div style="font-size:12px; opacity:.7; margin-top:10px;">
      Controls — Desktop: WASD to move, hold left mouse to fire. Mobile: drag left to move, tap/drag right to shoot.
    </div>
  </div>
</div>

<script>
(function(){
  const TAU = Math.PI * 2;
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const mini = document.getElementById('mini');
  const mctx = mini.getContext('2d');

  // Resize to device pixels for crispness
  function fitCanvas() {
    const dpr = Math.max(1, Math.min(2, window.devicePixelRatio || 1));
    canvas.width = Math.floor(window.innerWidth * dpr);
    canvas.height = Math.floor(window.innerHeight * dpr);
    canvas.style.width = '100vw';
    canvas.style.height = '100vh';
    ctx.setTransform(dpr,0,0,dpr,0,0); // draw in CSS pixels
  }
  window.addEventListener('resize', fitCanvas);
  fitCanvas();

  // Map
  const MAP_SIZE = 3000; // square world
  // Game state
  let state;

  function rand(a,b){ return a + Math.random()*(b-a); }
  function randi(a,b){ return Math.floor(rand(a,b)); }
  function clamp(v,a,b){ return Math.max(a, Math.min(b, v)); }
  function dist(ax,ay,bx,by){ const dx=ax-bx, dy=ay-by; return Math.hypot(dx,dy); }
  function angle(ax,ay,bx,by){ return Math.atan2(by-ay, bx-ax); }
  function lerp(a,b,t){ return a + (b-a)*t; }

  function spawnObstacles(n=60) {
    const obs = [];
    for (let i=0;i<n;i++){
      const w = rand(80, 220), h = rand(80, 220);
      const x = rand(w, MAP_SIZE - w);
      const y = rand(h, MAP_SIZE - h);
      obs.push({x,y,w,h});
    }
    return obs;
  }

  function spawnCrates(n=18) {
    const arr = [];
    for (let i=0;i<n;i++){
      arr.push({x:rand(200, MAP_SIZE-200), y:rand(200, MAP_SIZE-200), taken:false, kind: Math.random()<0.5?'ammo':'med'});
    }
    return arr;
  }

  function newBot(id) {
    const x = rand(300, MAP_SIZE-300), y = rand(300, MAP_SIZE-300);
    return {
      id, bot:true, x, y, r:14, hp:100, ammo: randi(30,90),
      aim:0, fireCd:0, speed: 2.0,
      targetTime:0, tx:x, ty:y, alive:true
    };
  }

  function newPlayer() {
    return {
      id:'player', bot:false, x: MAP_SIZE/2, y: MAP_SIZE/2,
      r:16, hp:100, ammo:60, aim:0, fireCd:0, speed: 2.6, alive:true, kills:0
    };
  }

  function reset() {
    state = {
      t:0, // seconds
      phase:1,
      bullets:[],
      obstacles: spawnObstacles(),
      crates: spawnCrates(),
      player: newPlayer(),
      bots: Array.from({length: 20}, (_,i)=>newBot(i)),
      // safe zone
      zone: {
        cx: rand(600, MAP_SIZE-600),
        cy: rand(600, MAP_SIZE-600),
        r: MAP_SIZE*0.65,
        targetR: MAP_SIZE*0.65,
        shrinkTimer: 10, // seconds to wait before shrink
        shrinkDur: 45, // shrink duration
        gasDps: 8
      },
      camera: {x:0,y:0, zoom:1},
      input: {up:false,down:false,left:false,right:false, mouse:{x:0,y:0, down:false}},
      over:false, win:false
    };
    // Next zone target for future phases
    pickNextZone();
    updateHud();
    banner(`Prepare to drop…`);
  }

  function pickNextZone() {
    const z = state.zone;
    const nx = clamp(z.cx + rand(-300,300), 400, MAP_SIZE-400);
    const ny = clamp(z.cy + rand(-300,300), 400, MAP_SIZE-400);
    const nr = Math.max(300, z.targetR * 0.65);
    z.next = {cx:nx, cy:ny, r:nr};
  }

  function startShrink() {
    const z = state.zone;
    z.shrinking = true;
    z.shrinkElapsed = 0;
    banner(`Zone is shrinking!`);
  }

  function endShrink() {
    const z = state.zone;
    z.shrinking = false;
    z.cx = z.next.cx; z.cy = z.next.cy; z.r = z.next.r;
    z.targetR = z.r; // reached
    state.phase++;
    document.getElementById('phase').textContent = state.phase;
    // prepare next
    pickNextZone();
    z.shrinkTimer = 12;
    z.shrinkDur = Math.max(22, z.shrinkDur*0.8);
    z.gasDps = Math.min(20, z.gasDps + 2);
    banner(`Phase ${state.phase}`);
  }

  function banner(text) {
    const el = document.getElementById('phaseText');
    el.textContent = text;
    el.style.opacity = 1;
    clearTimeout(el._t);
    el._t = setTimeout(()=>{ el.style.opacity = 0; }, 2200);
  }

  // Input
  const keys = {};
  window.addEventListener('keydown', e=>{ keys[e.code]=true; });
  window.addEventListener('keyup', e=>{ keys[e.code]=false; });

  canvas.addEventListener('mousemove', e=>{
    const rect = canvas.getBoundingClientRect();
    state.input.mouse.x = e.clientX - rect.left;
    state.input.mouse.y = e.clientY - rect.top;
  });
  canvas.addEventListener('mousedown', ()=>{ state.input.mouse.down = true; });
  window.addEventListener('mouseup', ()=>{ state.input.mouse.down = false; });

  // Simple mobile/touch controls
  let leftTouchId=null, rightTouchId=null;
  let leftStart=null, leftVec={x:0,y:0};
  canvas.addEventListener('touchstart', e=>{
    for (const t of e.changedTouches){
      if (t.clientX < window.innerWidth/2 && leftTouchId===null){
        leftTouchId = t.identifier; leftStart = {x:t.clientX, y:t.clientY};
      } else if (rightTouchId===null) {
        rightTouchId = t.identifier;
        state.input.mouse.down = true;
        state.input.mouse.x = t.clientX; state.input.mouse.y = t.clientY;
      }
    }
    e.preventDefault();
  }, {passive:false});
  canvas.addEventListener('touchmove', e=>{
    for (const t of e.changedTouches){
      if (t.identifier===leftTouchId && leftStart){
        leftVec.x = (t.clientX - leftStart.x)/60;
        leftVec.y = (t.clientY - leftStart.y)/60;
        leftVec.x = clamp(leftVec.x, -1, 1);
        leftVec.y = clamp(leftVec.y, -1, 1);
      }
      if (t.identifier===rightTouchId){
        state.input.mouse.x = t.clientX; state.input.mouse.y = t.clientY;
      }
    }
    e.preventDefault();
  }, {passive:false});
  window.addEventListener('touchend', e=>{
    for (const t of e.changedTouches){
      if (t.identifier===leftTouchId){ leftTouchId=null; leftStart=null; leftVec.x=leftVec.y=0; }
      if (t.identifier===rightTouchId){ rightTouchId=null; state.input.mouse.down=false; }
    }
  });

  function handleInput(dt) {
    const p = state.player;
    const {mouse} = state.input;
    const up = keys['KeyW'] || keys['ArrowUp'];
    const down = keys['KeyS'] || keys['ArrowDown'];
    const left = keys['KeyA'] || keys['ArrowLeft'];
    const right = keys['KeyD'] || keys['ArrowRight'];

    let vx = 0, vy = 0;
    if (up) vy -= 1;
    if (down) vy += 1;
    if (left) vx -= 1;
    if (right) vx += 1;
    // touch vector adds in
    vx += leftVec.x; vy += leftVec.y;

    const mag = Math.hypot(vx,vy) || 1;
    vx /= mag; vy /= mag;

    const s = p.speed * (p.hp>0?1:0);
    p.x = clamp(p.x + vx * s * 60 * dt, 0, MAP_SIZE);
    p.y = clamp(p.y + vy * s * 60 * dt, 0, MAP_SIZE);

    // update aim to mouse world position
    const wx = p.x - canvas.width/2 + mouse.x;
    const wy = p.y - canvas.height/2 + mouse.y;
    p.aim = angle(p.x, p.y, wx, wy);

    // firing
    if (mouse.down) tryShoot(p);
  }

  function tryShoot(ent){
    if (!ent.alive || ent.ammo<=0) return;
    if (ent.fireCd>0) return;
    const spread = ent.bot ? 0.14 : 0.06;
    const a = ent.aim + rand(-spread, spread);
    const speed = 820; // px/s
    state.bullets.push({
      x: ent.x, y: ent.y,
      vx: Math.cos(a)*speed, vy: Math.sin(a)*speed,
      life: 1.2, owner: ent.id
    });
    ent.fireCd = ent.bot ? rand(0.12,0.25) : 0.09;
    ent.ammo--;
    if (ent===state.player) document.getElementById('ammo').textContent = ent.ammo;
  }

  function updateBullets(dt){
    const obs = state.obstacles;
    for (let i=state.bullets.length-1;i>=0;i--){
      const b = state.bullets[i];
      b.life -= dt;
      if (b.life<=0){ state.bullets.splice(i,1); continue; }
      b.x += b.vx*dt; b.y += b.vy*dt;
      // hit walls
      if (b.x<0||b.x>MAP_SIZE||b.y<0||b.y>MAP_SIZE){ state.bullets.splice(i,1); continue; }
      // hit obstacles simple AABB
      for (const o of obs){
        if (b.x>o.x-o.w/2 && b.x<o.x+o.w/2 && b.y>o.y-o.h/2 && b.y<o.y+o.h/2){
          state.bullets.splice(i,1); continue;
        }
      }
      // hit actors
      if (hitEntity(state.player, b)){ state.bullets.splice(i,1); continue; }
      for (let j=state.bots.length-1;j>=0;j--){
        const bot = state.bots[j];
        if (!bot.alive) continue;
        if (hitEntity(bot, b)){ state.bullets.splice(i,1); break; }
      }
    }
  }

  function hitEntity(ent, bullet){
    if (bullet.owner===ent.id || !ent.alive) return false;
    if (dist(ent.x, ent.y, bullet.x, bullet.y) < ent.r + 4){
      const dmg = randi(16,28);
      ent.hp -= dmg;
      if (ent.hp<=0){ ent.alive=false; ent.hp=0; onDeath(ent, bullet.owner); }
      if (ent===state.player) document.getElementById('hp').textContent = ent.hp;
      return true;
    }
    return false;
  }

  function onDeath(ent, killerId){
    if (ent===state.player){
      endGame(false, `You were eliminated. Killer: ${killerId}`);
    } else {
      // bot died
      if (killerId==='player'){ state.player.kills++; document.getElementById('kills').textContent = state.player.kills; }
      // drop a crate sometimes
      if (Math.random()<0.35){
        state.crates.push({x:ent.x, y:ent.y, taken:false, kind: Math.random()<0.6?'ammo':'med'});
      }
      checkWin();
    }
  }

  function checkWin(){
    const aliveBots = state.bots.filter(b=>b.alive).length;
    if (aliveBots===0 && state.player.alive){
      endGame(true, `K/D: ${state.player.kills} • Time: ${formatTime(state.t)}`);
    }
  }

  function botAI(dt){
    const p = state.player, z = state.zone;
    for (const b of state.bots){
      if (!b.alive) continue;
      b.fireCd = Math.max(0, b.fireCd - dt);
      b.targetTime -= dt;
      // choose target point
      if (b.targetTime<=0){
        const biasToCenter = 0.6;
        const tx = lerp(b.x, z.cx + rand(-z.r*0.4, z.r*0.4), biasToCenter);
        const ty = lerp(b.y, z.cy + rand(-z.r*0.4, z.r*0.4), biasToCenter);
        b.tx = clamp(tx, 60, MAP_SIZE-60);
        b.ty = clamp(ty, 60, MAP_SIZE-60);
        b.targetTime = rand(0.8, 1.6);
      }
      // move toward target
      const ang = angle(b.x, b.y, b.tx, b.ty);
      const vx = Math.cos(ang)*b.speed*60*dt;
      const vy = Math.sin(ang)*b.speed*60*dt;
      b.x = clamp(b.x + vx, 0, MAP_SIZE);
      b.y = clamp(b.y + vy, 0, MAP_SIZE);

      // detect player
      const d = dist(b.x,b.y,p.x,p.y);
      if (p.alive && d < 520){
        b.aim = angle(b.x,b.y,p.x,p.y);
        if (d<480 && Math.random()<0.7) tryShoot(b);
        // sometimes chase
        if (d>200 && Math.random()<0.2){ b.tx = p.x; b.ty = p.y; }
      }
    }
  }

  function inZone(x,y){
    const z = state.zone;
    return dist(x,y,z.cx,z.cy) <= z.r;
  }

  function updateZone(dt){
    const z = state.zone;
    if (!z.shrinking){
      z.shrinkTimer -= dt;
      if (z.shrinkTimer<=0) startShrink();
    } else {
      z.shrinkElapsed += dt;
      const t = clamp(z.shrinkElapsed / z.shrinkDur, 0, 1);
      z.cx = lerp(z.cx, z.next.cx, t);
      z.cy = lerp(z.cy, z.next.cy, t);
      z.r = lerp(z.r, z.next.r, t);
      if (t>=1){ endShrink(); }
    }

    // gas damage
    for (const ent of [state.player, ...state.bots]){
      if (!ent.alive) continue;
      if (!inZone(ent.x, ent.y)){
        ent.hp -= z.gasDps * dt;
        if (ent.hp<=0){ ent.hp=0; ent.alive=false; onDeath(ent, 'zone'); }
      }
    }
  }

  function updateCrates(){
    const all = [state.player, ...state.bots];
    for (const c of state.crates){
      if (c.taken) continue;
      for (const ent of all){
        if (!ent.alive) continue;
        if (dist(ent.x,ent.y,c.x,c.y)<26){
          c.taken = true;
          if (c.kind==='ammo'){ ent.ammo += randi(15,40); if (ent===state.player) document.getElementById('ammo').textContent = ent.ammo; }
          else { ent.hp = clamp(ent.hp + randi(25,45), 0, 100); if (ent===state.player) document.getElementById('hp').textContent = ent.hp; }
        }
      }
    }
  }

  function collideObstacles(){
    const actors = [state.player, ...state.bots];
    for (const a of actors){
      if (!a.alive) continue;
      for (const o of state.obstacles){
        const dx = a.x - clamp(a.x, o.x - o.w/2, o.x + o.w/2);
        const dy = a.y - clamp(a.y, o.y - o.h/2, o.y + o.h/2);
        const overlap = a.r - Math.hypot(dx,dy);
        if (overlap>0){
          const len = Math.hypot(dx,dy) || 1;
          a.x += (dx/len) * overlap;
          a.y += (dy/len) * overlap;
        }
      }
    }
  }

  function update(dt){
    if (state.over) return;
    state.t += dt;
    handleInput(dt);
    for (const ent of [state.player, ...state.bots]){ ent.fireCd = Math.max(0, ent.fireCd - dt); }
    botAI(dt);
    updateBullets(dt);
    updateZone(dt);
    updateCrates();
    collideObstacles();
    updateHud();
    checkWin();
  }

  function updateHud(){
    document.getElementById('hp').textContent = Math.round(state.player.hp);
    document.getElementById('ammo').textContent = state.player.ammo|0;
    document.getElementById('kills').textContent = state.player.kills|0;
    document.getElementById('alive').textContent = (state.bots.filter(b=>b.alive).length + (state.player.alive?1:0));
    document.getElementById('time').textContent = formatTime(state.t);
  }

  function formatTime(t){
    const m = Math.floor(t/60), s = Math.floor(t%60);
    return `${m}:${s.toString().padStart(2,'0')}`;
  }

  function draw(){
    // camera centers on player
    const p = state.player;
    state.camera.x = clamp(p.x - canvas.width/2, 0, MAP_SIZE - canvas.width);
    state.camera.y = clamp(p.y - canvas.height/2, 0, MAP_SIZE - canvas.height);

    ctx.clearRect(0,0,canvas.width,canvas.height);

    // background grid
    drawGrid();

    // zone
    drawZone();

    // obstacles
    for (const o of state.obstacles) drawObstacle(o);

    // crates
    for (const c of state.crates) if (!c.taken) drawCrate(c);

    // bullets
    for (const b of state.bullets) drawBullet(b);

    // bots
    for (const b of state.bots) if (b.alive) drawActor(b, '#ff5a5f');

    // player
    if (p.alive) drawActor(p, '#66ffa8', true);

    drawMiniMap();
  }

  function worldToScreen(wx,wy){
    return { x: wx - state.camera.x, y: wy - state.camera.y };
  }

  function drawGrid(){
    const sz = 80;
    const ox = - (state.camera.x % sz);
    const oy = - (state.camera.y % sz);
    ctx.globalAlpha = 1;
    ctx.fillStyle = '#0b0e11';
    ctx.fillRect(0,0,canvas.width, canvas.height);
    ctx.strokeStyle = 'rgba(255,255,255,0.05)';
    ctx.lineWidth = 1;
    for (let x=ox; x<canvas.width; x+=sz){
      ctx.beginPath(); ctx.moveTo(x,0); ctx.lineTo(x,canvas.height); ctx.stroke();
    }
    for (let y=oy; y<canvas.height; y+=sz){
      ctx.beginPath(); ctx.moveTo(0,y); ctx.lineTo(canvas.width,y); ctx.stroke();
    }
  }

  function drawZone(){
    const z = state.zone;
    const s = worldToScreen(z.cx, z.cy);
    // outer gas dark
    ctx.beginPath();
    ctx.arc(s.x, s.y, z.r, 0, TAU);
    ctx.strokeStyle = 'rgba(93, 187, 255, 0.9)';
    ctx.lineWidth = 3;
    ctx.stroke();

    // next zone hint
    if (z.next){
      ctx.beginPath();
      const n = worldToScreen(z.next.cx, z.next.cy);
      ctx.setLineDash([6,6]);
      ctx.arc(n.x, n.y, z.next.r, 0, TAU);
      ctx.strokeStyle = 'rgba(255,255,255,0.25)';
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.setLineDash([]);
    }
  }

  function drawObstacle(o){
    const s = worldToScreen(o.x, o.y);
    ctx.fillStyle = '#1b2229';
    ctx.strokeStyle = 'rgba(255,255,255,0.06)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.rect(s.x - o.w/2, s.y - o.h/2, o.w, o.h);
    ctx.fill(); ctx.stroke();
  }

  function drawCrate(c){
    const s = worldToScreen(c.x, c.y);
    ctx.fillStyle = c.kind==='ammo' ? '#5aa2ff' : '#6aff9f';
    ctx.strokeStyle = 'rgba(0,0,0,0.4)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.rect(s.x-8, s.y-8, 16, 16);
    ctx.fill(); ctx.stroke();
  }

  function drawBullet(b){
    const s = worldToScreen(b.x, b.y);
    ctx.fillStyle = '#ffd166';
    ctx.beginPath();
    ctx.arc(s.x, s.y, 3, 0, TAU);
    ctx.fill();
  }

  function drawActor(a, color, isPlayer=false){
    const s = worldToScreen(a.x, a.y);
    // body
    ctx.fillStyle = color;
    ctx.beginPath();
    ctx.arc(s.x, s.y, a.r, 0, TAU);
    ctx.fill();

    // barrel line
    ctx.strokeStyle = 'rgba(0,0,0,0.7)';
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.moveTo(s.x, s.y);
    ctx.lineTo(s.x + Math.cos(a.aim)* (a.r+8), s.y + Math.sin(a.aim)*(a.r+8));
    ctx.stroke();

    // health bar
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(s.x-18, s.y - a.r - 12, 36, 6);
    ctx.fillStyle = '#7cff6a';
    ctx.fillRect(s.x-18, s.y - a.r - 12, 36*(a.hp/100), 6);
    if (isPlayer){
      // crosshair
      const mx = state.input.mouse.x, my = state.input.mouse.y;
      ctx.strokeStyle = 'rgba(255,255,255,0.5)'; ctx.lineWidth = 1;
      ctx.beginPath(); ctx.arc(mx, my, 10, 0, TAU); ctx.stroke();
    }
  }

  function drawMiniMap(){
    const scale = mini.width / MAP_SIZE;
    mctx.clearRect(0,0,mini.width, mini.height);
    // background
    mctx.fillStyle = '#0b0e11'; mctx.fillRect(0,0,mini.width, mini.height);
    // zone
    mctx.strokeStyle = 'rgba(93, 187, 255, 0.9)'; mctx.lineWidth = 2;
    mctx.beginPath();
    mctx.arc(state.zone.cx*scale, state.zone.cy*scale, state.zone.r*scale, 0, TAU);
    mctx.stroke();
    if (state.zone.next){
      mctx.setLineDash([4,4]);
      mctx.strokeStyle = 'rgba(255,255,255,0.25)';
      mctx.beginPath();
      mctx.arc(state.zone.next.cx*scale, state.zone.next.cy*scale, state.zone.next.r*scale, 0, TAU);
      mctx.stroke();
      mctx.setLineDash([]);
    }
    // player
    mctx.fillStyle = '#66ffa8';
    mctx.beginPath();
    mctx.arc(state.player.x*scale, state.player.y*scale, 3, 0, TAU);
    mctx.fill();
    // bots
    mctx.fillStyle = '#ff5a5f';
    for (const b of state.bots) if (b.alive){
      mctx.beginPath();
      mctx.arc(b.x*scale, b.y*scale, 2.5, 0, TAU);
      mctx.fill();
    }
    // viewport rectangle
    const viewW = canvas.width * (1/window.devicePixelRatio || 1);
    const viewH = canvas.height * (1/window.devicePixelRatio || 1);
    mctx.strokeStyle = 'rgba(255,255,255,0.35)';
    mctx.strokeRect(state.camera.x*scale, state.camera.y*scale, viewW*scale, viewH*scale);
  }

  // Game loop
  let last = performance.now();
  function frame(now){
    const dt = Math.min(0.033, (now - last)/1000);
    last = now;
    update(dt);
    draw();
    requestAnimationFrame(frame);
  }

  function endGame(win, subtitle){
    state.over = true; state.win = win;
    const m = document.getElementById('modal');
    document.getElementById('modalTitle').textContent = win ? 'Winner Winner!' : 'Game Over';
    document.getElementById('modalSubtitle').textContent = subtitle || '';
    m.style.display = 'flex';
  }

  document.getElementById('playBtn').addEventListener('click', ()=>{
    document.getElementById('modal').style.display = 'none';
    reset();
  });

  // Kick off
  reset();
  requestAnimationFrame(frame);
})();
</script>
</body>
</html># Game
