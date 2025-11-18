<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Asteroid Shooter Pro</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <!-- Tailwind for UI -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- Three.js -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <!-- Cannon.js -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/cannon.js/0.6.2/cannon.min.js"></script>

  <style>
    html, body { margin:0; padding:0; background:#000; color:#e8f0ff; font-family:Inter, system-ui, -apple-system, Segoe UI; overflow:hidden; }
    canvas { display:block; }
    #hud { position:fixed; top:8px; left:8px; right:8px; z-index:10; display:flex; justify-content:space-between; align-items:center; gap:12px; padding:8px 12px; }
    .panel { background:#111827cc; border:1px solid #1f2937; border-radius:10px; padding:8px 12px; text-shadow:0 0 4px #00ffff88; }
    #controls { position:fixed; bottom:12px; left:50%; transform:translateX(-50%); z-index:10; display:flex; gap:8px; }
    button { background:#1f2937; color:#e8f0ff; border:1px solid #374151; border-radius:999px; padding:10px 16px; font-weight:700; cursor:pointer; }
    button:hover { background:#374151; box-shadow:0 0 10px rgba(0,255,255,0.35); transform:translateY(-1px); }
    #modal, #missionModal, #overModal { position:fixed; inset:0; display:none; align-items:center; justify-content:center; background:rgba(0,0,0,0.7); z-index:20; }
    .modalCard { background:#0f172acc; border:1px solid #334155; border-radius:12px; padding:18px; width:900px; max-width:92vw; }
    .grid { display:grid; gap:10px; grid-template-columns:repeat(2, minmax(0,1fr)); }
    .card { border:1px solid #334155; border-radius:10px; padding:12px; }
    .shipCard { transition:transform 0.2s; }
    .shipCard:hover { transform:scale(1.03); box-shadow:0 0 16px rgba(0,255,255,0.4); }
    #status { position:fixed; top:12px; right:12px; background:#111827; border:1px solid #334155; padding:8px 12px; border-radius:8px; opacity:0; transition:opacity 0.3s; z-index:25; }
  </style>
</head>
<body>
  <!-- HUD -->
  <div id="hud">
    <div class="panel flex gap-12">
      <div><strong>Level:</strong> <span id="level">1</span></div>
      <div><strong>Score:</strong> <span id="score">0</span></div>
      <div><strong>Lives:</strong> <span id="lives">3</span></div>
      <div><strong>Heat:</strong> <span id="heat">0%</span></div>
      <div><strong>Combo:</strong> <span id="combo">x1.0</span></div>
      <div><strong>Ship:</strong> <span id="shipName">Standard Scout</span></div>
      <div><strong>Mode:</strong> <span id="modeName">Arcade</span></div>
      <div><strong>Objective:</strong> <span id="objective">Earn points</span></div>
    </div>
    <div class="panel">
      <strong>Controls:</strong> WASD/Arrows to move • Space to fire • Shift to dash • P pause • H hangar • M missions • S save
    </div>
  </div>

  <!-- Controls -->
  <div id="controls">
    <button id="saveBtn">Save</button>
    <button id="hangarBtn">Shipyard</button>
    <button id="pauseBtn">Pause</button>
    <button id="modeBtn">Switch Mode</button>
    <button id="missionBtn">Missions</button>
  </div>

  <!-- Modals -->
  <div id="modal"><div class="modalCard">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-2xl font-bold">Shipyard hangar</h2>
      <button id="closeHangar">Close</button>
    </div>
    <div class="mb-3"><strong>Credits:</strong> <span id="credits">0</span> SCORE</div>
    <div id="shipGrid" class="grid"></div>
  </div></div>

  <div id="missionModal"><div class="modalCard">
    <div class="flex justify-between items-center mb-2">
      <h2 class="text-2xl font-bold">Mission briefing</h2>
      <button id="closeMission">Close</button>
    </div>
    <div id="missionDesc" class="grid"></div>
    <div class="mt-3"><button id="startMission" class="w-full">Start mission</button></div>
  </div></div>

  <div id="overModal"><div class="modalCard">
    <h2 class="text-2xl font-bold mb-2">Game over</h2>
    <p class="mb-4">Final score: <strong id="finalScore">0</strong></p>
    <div class="flex gap-8">
      <button id="retryBtn">Retry</button>
      <button id="goHangar">Shipyard</button>
    </div>
  </div></div>

  <!-- Status toast -->
  <div id="status">Saved!</div>

  <script>
    // ---------- Audio ----------
    let audioCtx; const sfx = {
      init: () => { audioCtx = audioCtx || new (window.AudioContext || window.webkitAudioContext)(); },
      beep(freq=440, time=0.08, type='square', vol=0.12) {
        if (!audioCtx) return;
        const o = audioCtx.createOscillator(), g = audioCtx.createGain();
        o.type = type; o.frequency.value = freq * (0.92 + Math.random()*0.16);
        g.gain.value = vol; o.connect(g).connect(audioCtx.destination);
        o.start(); setTimeout(()=>o.stop(), time*1000);
      },
      shoot: () => sfx.beep(760,0.05,'square',0.10),
      hit: () => sfx.beep(200,0.06,'sawtooth',0.12),
      explode: () => { sfx.beep(120,0.12,'triangle',0.18); setTimeout(()=>sfx.beep(80,0.1,'sine',0.12),80); },
      level: () => { sfx.beep(600,0.08,'sine',0.12); setTimeout(()=>sfx.beep(800,0.08,'sine',0.12),90); },
      over: () => { sfx.beep(220,0.15,'triangle',0.15); setTimeout(()=>sfx.beep(140,0.2,'sine',0.12),160); }
    };
    addEventListener('click', ()=>sfx.init(), { once:true });

    // ---------- Physics & 3D ----------
    let scene, camera, renderer, world;
    const clock = new THREE.Clock();
    const objects = []; // pooled enemies/asteroids
    const bullets = [];
    const particles = [];

    function initPhysics() {
      world = new CANNON.World();
      world.gravity.set(0,0,0);
      world.broadphase = new CANNON.NaiveBroadphase();
      world.solver.iterations = 10;
    }
    function initScene() {
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(70, innerWidth/innerHeight, 0.1, 1000);
      camera.position.z = 16;

      renderer = new THREE.WebGLRenderer({ antialias:true });
      renderer.setSize(innerWidth, innerHeight);
      renderer.setClearColor(0x000000);
      renderer.shadowMap.enabled = true;
      document.body.appendChild(renderer.domElement);

      const dir = new THREE.DirectionalLight(0xffffff, 0.9); dir.position.set(20,30,10); dir.castShadow = true; scene.add(dir);
      const amb = new THREE.AmbientLight(0x404040, 0.6); scene.add(amb);
      addStarfield();
    }
    function addStarfield() {
      const g = new THREE.BufferGeometry(), stars = 800, positions = new Float32Array(stars*3);
      for (let i=0;i<stars;i++){ positions[i*3+0]=(Math.random()-0.5)*60; positions[i*3+1]=(Math.random()-0.5)*40; positions[i*3+2]=(Math.random()-0.5)*10; }
      g.setAttribute('position', new THREE.BufferAttribute(positions,3));
      const m = new THREE.PointsMaterial({ color:0x9ca3af, size:0.04 });
      const points = new THREE.Points(g,m); scene.add(points);
    }

    // ---------- Player ----------
    const JETS = {
      1:{ name:'Standard Scout', color:0x00ffff, speed: 10, damage:1, lives:3, perk:'scoreBoost' },
      2:{ name:'Heavy Defender', color:0x00ff00, speed: 7.5, damage:1.6, lives:5, perk:'shieldOnHit' },
      3:{ name:'Falcon Striker', color:0xff0000, speed: 14, damage:1.8, lives:3, perk:'rapidFire' },
      4:{ name:'Mining Drill', color:0xffa500, speed: 6, damage:2.3, lives:4, perk:'pierce' },
      5:{ name:'Stealth Runner', color:0x800080, speed: 16, damage:0.8, lives:2, perk:'cloak' },
      6:{ name:'Nebula Cruiser', color:0xda70d6, speed: 11, damage:1.2, lives:4, perk:'pierce' },
      7:{ name:'Solar Harvester', color:0xf0e68c, speed: 10, damage:1.7, lives:5, perk:'regen' },
      8:{ name:'Quantum Jumper', color:0x4169e1, speed: 18, damage:1.0, lives:3, perk:'dash' },
      9:{ name:'Void Sentinel', color:0x3cb371, speed: 11, damage:2.0, lives:6, perk:'drones' },
      10:{ name:'The Dreadnought', color:0x8b0000, speed: 6, damage:2.8, lives:10, perk:'bossSlayer' }
    };
    const costs = {1:0,2:2000,3:5000,4:7500,5:12000,6:15000,7:20000,8:30000,9:50000,10:100000};

    const player = {
      mesh:null, body:null,
      vx:0, vy:0, fireCD:0, heat:0, overheated:false, lives:3,
      shipId:1, damage:1, speed:10, combo:1, invuln:0, drones:[]
    };

    function createPlayer() {
      const geo = new THREE.ConeGeometry(0.6, 1.8, 10);
      const mat = new THREE.MeshStandardMaterial({ color:JETS[1].color, metalness:0.8, roughness:0.3 });
      player.mesh = new THREE.Mesh(geo, mat);
      player.mesh.rotation.x = Math.PI/2;
      player.mesh.castShadow = true; player.mesh.receiveShadow = true;
      scene.add(player.mesh);

      const shape = new CANNON.Sphere(0.7);
      player.body = new CANNON.Body({ mass:5, shape });
      player.body.position.set(0,0,0);
      player.body.linearDamping = 0.92; player.body.angularDamping = 0.92;
      world.addBody(player.body);

      equipShip(1);
    }

    function equipShip(id) {
      const s = JETS[id]; if (!s) return;
      player.shipId = id; player.speed = s.speed; player.damage = s.damage;
      player.lives = Math.max(player.lives, s.lives);
      player.mesh.material.color.set(s.color);
      player.drones = [];
      if (s.perk==='drones') for (let i=0;i<2;i++) player.drones.push({ angle:i*Math.PI, cd:0 });
      flashStatus(`Equipped: ${s.name}`);
      updateHUD();
    }

    // ---------- HUD & State ----------
    const levelEl = document.getElementById('level');
    const scoreEl = document.getElementById('score');
    const livesEl = document.getElementById('lives');
    const heatEl = document.getElementById('heat');
    const comboEl = document.getElementById('combo');
    const shipNameEl = document.getElementById('shipName');
    const modeNameEl = document.getElementById('modeName');
    const objectiveEl = document.getElementById('objective');
    const creditsEl = document.getElementById('credits');

    const state = {
      score:0, level:1, mode:'arcade',
      mission:{ active:false, id:null, desc:'', timer:0, kills:0, boss:false, cargo:null },
      paused:false, over:false, lastSpawn:0
    };
    const unlocked = new Set([1]);

    function updateHUD() {
      levelEl.textContent = state.level;
      scoreEl.textContent = Math.floor(state.score);
      livesEl.textContent = player.lives;
      heatEl.textContent = `${Math.floor(player.heat*100)}%`;
      comboEl.textContent = `x${player.combo.toFixed(1)}`;
      shipNameEl.textContent = JETS[player.shipId].name;
      modeNameEl.textContent = state.mode[0].toUpperCase()+state.mode.slice(1);
      objectiveEl.textContent = state.mission.active ? state.mission.desc : (state.mode==='arcade'?'Earn points':'Choose mission');
      creditsEl.textContent = Math.floor(state.score);
    }

    // ---------- Enemies & bullets ----------
    function spawnEnemy(type=null) {
      const t = type || (state.level%5===0? 'boss' : ['asteroid','drone','turret','carrier'][Math.floor(Math.random()*4)]);
      const x = (Math.random()-0.5)*30, y = (Math.random()-0.5)*20;
      let mesh, body, hp=3, speed=3;
      if (t==='asteroid') {
        mesh = new THREE.Mesh(new THREE.DodecahedronGeometry(Math.random()*0.8+0.6), new THREE.MeshStandardMaterial({ color:0x6b7280, metalness:0.2 }));
        hp = 2 + Math.random()*2; speed = 2 + Math.random()*2;
      } else if (t==='drone') {
        mesh = new THREE.Mesh(new THREE.SphereGeometry(0.5,16,16), new THREE.MeshStandardMaterial({ color:0xf87171 }));
        hp = 2.5; speed = 4.5;
      } else if (t==='turret') {
        mesh = new THREE.Mesh(new THREE.BoxGeometry(0.9,0.9,0.9), new THREE.MeshStandardMaterial({ color:0xfb7185 }));
        hp = 4; speed = 0.5;
      } else if (t==='carrier') {
        mesh = new THREE.Mesh(new THREE.TorusKnotGeometry(0.5,0.2,64,8), new THREE.MeshStandardMaterial({ color:0xf59e0b }));
        hp = 8; speed = 1.2;
      } else {
        mesh = new THREE.Mesh(new THREE.SphereGeometry(1.2,20,20), new THREE.MeshStandardMaterial({ color:0xef4444 }));
        hp = 70 + state.level*10; speed = 1.0;
      }
      mesh.castShadow = true; mesh.receiveShadow = true; scene.add(mesh);

      const shape = new CANNON.Sphere(0.6);
      body = new CANNON.Body({ mass: 3, shape });
      body.position.set(x, y, 0);
      world.addBody(body);

      objects.push({ type:t, mesh, body, hp, cd:0, alive:true });
    }

    function shootBullet(angle, speed=18, damage=player.damage, enemy=false) {
      const bGeo = new THREE.SphereGeometry(0.12, 8, 8);
      const bMat = new THREE.MeshStandardMaterial({ color: enemy?0xfbbf24:0x93c5fd, emissive: enemy?0x7c3aed:0x1e3a8a, emissiveIntensity:0.6 });
      const mesh = new THREE.Mesh(bGeo,bMat); mesh.castShadow=true;
      scene.add(mesh);

      const shape = new CANNON.Sphere(0.1);
      const body = new CANNON.Body({ mass:0.1, shape });
      body.position.copy(enemy? (objects.find(o=>o.type==='turret')?.body.position || new CANNON.Vec3(0,0,0)) : player.body.position);
      world.addBody(body);

      body.velocity.set(Math.cos(angle)*speed, Math.sin(angle)*speed, 0);
      bullets.push({ mesh, body, dmg:damage, enemy, alive:true, pierce: JETS[player.shipId].perk==='pierce' ? 1 : 0 });
      sfx.shoot();
    }

    function damagePlayer(n=1) {
      if (player.invuln>0) return;
      player.lives -= n; player.combo=1; player.invuln=0.6;
      sfx.hit(); screenShake(10, 160); burst(player.mesh.position.x, player.mesh.position.y, '#fde68a', 14);
      if (player.lives<=0) endGame();
      updateHUD();
    }

    function killEnemy(o) {
      o.alive=false; sfx.explode();
      const base = o.type==='boss'?1200:(o.type==='carrier'?300:120);
      let gain = base;
      if (JETS[player.shipId].perk==='scoreBoost') gain *= 1.10;
      player.combo = Math.min(3.5, player.combo + 0.12);
      gain *= player.combo; state.score += gain;
      if (o.type==='boss' && state.mission.active) state.mission.boss = true;
      burst(o.mesh.position.x, o.mesh.position.y, '#f87171', o.type==='boss'?60:24);
      screenShake(o.type==='boss'?16:8, o.type==='boss'?240:140);
      updateHUD();
    }

    // ---------- Effects ----------
    const shake = { t:0, mag:0 };
    function screenShake(mag=6, dur=160){ shake.t = dur; shake.mag = mag; }
    function burst(x,y,color='#93c5fd',count=14){
      for (let i=0;i<count;i++){
        const g = new THREE.SphereGeometry(0.06,6,6);
        const m = new THREE.MeshStandardMaterial({ color });
        const p = new THREE.Mesh(g,m); p.position.set(x,y,0);
        scene.add(p); particles.push({ mesh:p, vx:(Math.random()-0.5)*8, vy:(Math.random()-0.5)*8, life:0.8 });
      }
    }

    // ---------- Input ----------
    const keys = {};
    addEventListener('keydown', e => {
      const k = e.key.toLowerCase(); keys[k]=true;
      if (k==='p') togglePause();
      if (k==='h') openHangar();
      if (k==='m') openMission();
      if (k==='s') saveGame();
    });
    addEventListener('keyup', e => { keys[e.key.toLowerCase()]=false; });

    // ---------- UI handlers ----------
    document.getElementById('saveBtn').addEventListener('click', saveGame);
    document.getElementById('hangarBtn').addEventListener('click', openHangar);
    document.getElementById('pauseBtn').addEventListener('click', togglePause);
    document.getElementById('modeBtn').addEventListener('click', () => switchMode());
    document.getElementById('missionBtn').addEventListener('click', openMission);
    document.getElementById('closeHangar').addEventListener('click', closeHangar);
    document.getElementById('closeMission').addEventListener('click', closeMission);
    document.getElementById('startMission').addEventListener('click', () => { state.mode='mission'; closeMission(); updateHUD(); });
    document.getElementById('retryBtn').addEventListener('click', ()=>{ document.getElementById('overModal').style.display='none'; restart(); });
    document.getElementById('goHangar').addEventListener('click', ()=>{ document.getElementById('overModal').style.display='none'; openHangar(); });

    function flashStatus(msg='Saved!', time=1600){
      const el = document.getElementById('status'); el.textContent = msg; el.style.opacity = 1;
      setTimeout(()=>el.style.opacity=0, time);
    }

    // ---------- Hangar ----------
    function openHangar(){
      const grid = document.getElementById('shipGrid'); grid.innerHTML = '';
      Object.entries(JETS).forEach(([id,s])=>{
        const price = costs[id]; const owned = unlocked.has(+id);
        const equipped = player.shipId===+id;
        const card = document.createElement('div'); card.className = 'card shipCard';
        card.innerHTML = `
          <div class="flex justify-between items-center">
            <h3 class="font-bold">${s.name}</h3><span>${equipped?'Equipped':owned?'Unlocked':'Locked'}</span>
          </div>
          <div class="text-sm mt-1">Speed: <span class="text-yellow-300">${s.speed.toFixed(1)}</span> • Damage: <span class="text-red-400">${s.damage.toFixed(1)}</span> • Lives: <span class="text-green-400">${s.lives}</span></div>
          ${!owned? `<div class="mt-2 text-yellow-400">Cost: ${price.toLocaleString()} SCORE</div>`:''}
          <div class="mt-2 flex gap-8">
            ${equipped ? `<button disabled>Currently equipped</button>` :
              owned ? `<button data-equip="${id}">Equip</button>` :
              `<button data-buy="${id}">Buy now</button>`}
          </div>
        `;
        grid.appendChild(card);
      });
      document.getElementById('modal').style.display='flex';
      updateHUD();

      document.querySelectorAll('[data-equip]').forEach(btn=>{
        btn.addEventListener('click', ()=>{ equipShip(+btn.dataset.equip); closeHangar(); });
      });
      document.querySelectorAll('[data-buy]').forEach(btn=>{
        btn.addEventListener('click', ()=>{
          const id = +btn.dataset.buy, price = costs[id];
          if (state.score >= price) { state.score -= price; unlocked.add(id); equipShip(id); flashStatus(`Unlocked: ${JETS[id].name}`); }
          else flashStatus(`Need ${(price-state.score).toLocaleString()} more score`);
          openHangar();
        });
      });
    }
    function closeHangar(){ document.getElementById('modal').style.display='none'; }

    // ---------- Missions ----------
    const missionList = [
      { id:'survive60', name:'Survive 60s', desc:'Endure a relentless onslaught for 60 seconds.', start:()=>{ state.mission.timer=60; }, win:()=> state.mission.timer<=0 },
      { id:'kill30', name:'Destroy 30 enemies', desc:'Eliminate 30 hostiles.', start:()=>{ state.mission.kills=0; }, win:()=> state.mission.kills>=30 },
      { id:'bossHunt', name:'Defeat a boss', desc:'Destroy a sector boss.', start:()=>{ state.mission.boss=false; }, win:()=> state.mission.boss },
      { id:'protect', name:'Protect cargo', desc:'Defend a slow cargo drone for 45s.', start:()=>{ state.mission.timer=45; state.mission.cargo = spawnCargo(); }, win:()=> state.mission.timer<=0 && state.mission.cargo && state.mission.cargo.hp>0 }
    ];
    function openMission(){
      const grid = document.getElementById('missionDesc'); grid.innerHTML = '';
      missionList.forEach(m=>{
        const div = document.createElement('div'); div.className='card';
        div.innerHTML = `<div class="font-bold">${m.name}</div><div class="text-sm">${m.desc}</div><div class="mt-2"><button data-mid="${m.id}">Select</button></div>`;
        grid.appendChild(div);
      });
      document.getElementById('missionModal').style.display='flex';
      document.querySelectorAll('[data-mid]').forEach(btn=>{
        btn.addEventListener('click', ()=>{
          const m = missionList.find(x=>x.id===btn.dataset.mid);
          state.mission.active=true; state.mission.id=m.id; state.mission.desc=m.desc; m.start(); updateHUD(); flashStatus(`Mission: ${m.name}`);
        });
      });
    }
    function closeMission(){ document.getElementById('missionModal').style.display='none'; }

    function spawnCargo(){
      const mesh = new THREE.Mesh(new THREE.BoxGeometry(1.4,0.6,0.6), new THREE.MeshStandardMaterial({ color:0xf59e0b }));
      mesh.position.set(-12,0,0); scene.add(mesh);
      return { mesh, hp:30, vx:2.2 };
    }

    // ---------- Save/Load (localStorage fallback) ----------
    function saveGame(){
      const data = {
        score:state.score, level:state.level,
        lives:player.lives, shipId:player.shipId,
        unlocked:Array.from(unlocked)
      };
      localStorage.setItem('asteroidShooterSave', JSON.stringify(data));
      flashStatus('Game saved');
    }
    function loadGame(){
      const raw = localStorage.getItem('asteroidShooterSave'); if (!raw) return false;
      try {
        const d = JSON.parse(raw);
        state.score = d.score||0; state.level = d.level||1; player.lives = d.lives||JETS[d.shipId||1].lives;
        unlocked.clear(); (d.unlocked||[1]).forEach(x=>unlocked.add(x));
        equipShip(d.shipId||1);
        updateHUD();
        flashStatus('Game loaded');
        return true;
      } catch { return false; }
    }

    // ---------- Pause/Mode ----------
    function togglePause(){ state.paused = !state.paused; flashStatus(state.paused?'Paused':'Resumed'); }
    function switchMode(){ state.mode = state.mode==='arcade'?'mission':'arcade'; updateHUD(); flashStatus(`Mode: ${state.mode}`); }

    function endGame(){
      state.over = true; state.paused = true;
      document.getElementById('finalScore').textContent = Math.floor(state.score);
      document.getElementById('overModal').style.display='flex';
      sfx.over();
    }
    function restart(){
      state.score=0; state.level=1; player.lives=JETS[player.shipId].lives; player.combo=1; player.invuln=0; player.heat=0; player.overheated=false;
      objects.splice(0,objects.length); bullets.splice(0,bullets.length); particles.splice(0,particles.length);
      state.over=false; state.paused=false; state.mission.active=false; state.mission.timer=0; state.mission.kills=0; state.mission.boss=false;
      player.body.position.set(0,0,0); player.body.velocity.setZero();
      updateHUD(); flashStatus('Ready!');
    }

    // ---------- Game loop ----------
    function update(dt){
      // Movement
      const ax = ((keys['d']||keys['arrowright'])?1:0) - ((keys['a']||keys['arrowleft'])?1:0);
      const ay = ((keys['s']||keys['arrowdown'])?1:0) - ((keys['w']||keys['arrowup'])?1:0);
      player.body.velocity.x = ax * player.speed;
      player.body.velocity.y = ay * player.speed;
      player.mesh.position.copy(player.body.position);

      // Dash
      if ((keys['shift']) && JETS[player.shipId].perk==='dash' && player.invuln<=0) {
        player.body.position.x += Math.sign(player.body.velocity.x||1) * 1.2;
        player.body.position.y += Math.sign(player.body.velocity.y||1) * 1.2;
        player.invuln = 0.35; screenShake(10, 160); flashStatus('Dash!');
      }

      // Fire
      player.fireCD -= dt;
      const firing = keys[' '] || keys['f'];
      if (firing && player.fireCD<=0 && !player.overheated) {
        const rate = (JETS[player.shipId].perk==='rapidFire'? 12 : 8);
        player.fireCD = 1/rate;
        const angle = Math.atan2(player.body.velocity.y, player.body.velocity.x) || 0;
        if (player.shipId===10) { [-0.24,-0.12,0,0.12,0.24].forEach(off=>shootBullet(angle+off, 20, player.damage, false)); }
        else shootBullet(angle, 22, player.damage, false);
        player.heat = Math.min(1, player.heat + 0.05);
        if (player.heat>=1) player.overheated = true;
      }
      player.heat = Math.max(0, player.heat - (player.overheated ? 0.25*dt : 0.5*dt));
      if (player.overheated && player.heat<=0.25) player.overheated=false;
      player.invuln = Math.max(0, player.invuln - dt);

      // Drones
      if (JETS[player.shipId].perk==='drones') {
        player.drones.forEach(d=>{ d.angle += dt*2; d.cd -= dt; if (d.cd<=0){ d.cd=0.42; shootBullet(d.angle, 18, 0.6, false); } });
      }

      // Spawn logic
      state.lastSpawn += dt;
      const need = state.mode==='arcade'? (6 + state.level) : 6;
      if (state.lastSpawn>0.5 && objects.length<need) { spawnEnemy(); state.lastSpawn=0; }

      // Enemy AI
      objects.forEach(o=>{
        if (!o.alive) return;
        const dx = player.body.position.x - o.body.position.x;
        const dy = player.body.position.y - o.body.position.y;
        const ang = Math.atan2(dy, dx), dist = Math.hypot(dx, dy);
        if (o.type==='asteroid'){ o.body.velocity.x = Math.cos(ang)*2; o.body.velocity.y = Math.sin(ang)*2; }
        if (o.type==='drone'){ o.body.velocity.x = Math.cos(ang)*4; o.body.velocity.y = Math.sin(ang)*4; }
        if (o.type==='turret'){
          if (dist < 6) { o.body.velocity.x = -Math.cos(ang)*1.5; o.body.velocity.y = -Math.sin(ang)*1.5; }
          o.cd -= dt; if (o.cd<=0){ o.cd = 1.2; shootBullet(ang, 18, 0.7, true); }
        }
        if (o.type==='carrier'){
          o.cd -= dt; if (o.cd<=0){ o.cd=2.4; spawnEnemy('drone'); }
        }
        if (o.type==='boss'){
          o.cd -= dt; if (o.cd<=0){ o.cd=0.8; for(let i=0;i<8;i++){ const a=i*(Math.PI*2/8); shootBullet(a, 18, 0.9, true);} }
          o.body.velocity.x = Math.cos(ang)*1.2; o.body.velocity.y = Math.sin(ang)*1.2;
        }
      });

      // Bullets
      bullets.forEach(b=>{
        if (!b.alive) return;
        b.mesh.position.copy(b.body.position);
        // Offscreen
        if (Math.abs(b.body.position.x)>30 || Math.abs(b.body.position.y)>20){ b.alive=false; scene.remove(b.mesh); world.removeBody(b.body); }
        if (b.enemy){
          const dx = b.body.position.x - player.body.position.x, dy = b.body.position.y - player.body.position.y;
          if (dx*dx + dy*dy < 0.6*0.6){ b.alive=false; scene.remove(b.mesh); world.removeBody(b.body); damagePlayer(1); }
        } else {
          objects.forEach(o=>{
            if (!o.alive) return;
            const dx = b.body.position.x - o.body.position.x, dy = b.body.position.y - o.body.position.y;
            const r = o.type==='boss'?1.2:0.8;
            if (dx*dx + dy*dy < r*r){
              o.hp -= b.dmg; sfx.hit(); burst(b.body.position.x, b.body.position.y, '#fca5a5', 8);
              if (b.pierce>0) b.pierce--; else { b.alive=false; scene.remove(b.mesh); world.removeBody(b.body); }
              if (o.hp<=0){ killEnemy(o); scene.remove(o.mesh); world.removeBody(o.body); o.alive=false; state.mission.kills++; }
            }
          });
        }
      });

      // Particles
      particles.forEach(p=>{ p.mesh.position.x += p.vx*dt; p.mesh.position.y += p.vy*dt; p.life -= dt; if (p.life<=0){ scene.remove(p.mesh); } });

      // Leveling
      const needScore = state.level * 600;
      if (state.score > needScore){ state.level++; flashStatus(`Level ${state.level}`); sfx.level(); }

      // Missions
      if (state.mission.active){
        if (['survive60','protect'].includes(state.mission.id)) state.mission.timer = Math.max(0, state.mission.timer - dt);
        if (state.mission.id==='protect' && state.mission.cargo){
          const c = state.mission.cargo; c.mesh.position.x += c.vx*dt;
          if (Math.abs(c.mesh.position.x)>14) c.vx = -c.vx;
        }
        const win = (
          (state.mission.id==='survive60' && state.mission.timer<=0) ||
          (state.mission.id==='kill30' && state.mission.kills>=30) ||
          (state.mission.id==='bossHunt' && state.mission.boss) ||
          (state.mission.id==='protect' && state.mission.timer<=0 && state.mission.cargo && state.mission.cargo.hp>0)
        );
        if (win){ state.mission.active=false; flashStatus('Mission complete! +1000'); state.score += 1000; sfx.level(); updateHUD(); }
      }

      updateHUD();
    }

    // ---------- Animate ----------
    function animate(){
      requestAnimationFrame(animate);
      const dt = Math.min(0.033, clock.getDelta());
      if (!state.paused && !state.over){ world.step(1/60, dt, 3); update(dt); }
      // Shake camera subtly
      if (shake.t>0){ shake.t -= 16; camera.position.x = (Math.random()-0.5)*0.06*shake.mag; camera.position.y = (Math.random()-0.5)*0.06*shake.mag; } else { camera.position.x = 0; camera.position.y = 0; }
      renderer.render(scene,camera);
    }

    // ---------- Boot ----------
    function start(){
      initPhysics(); initScene(); createPlayer();
      loadGame(); updateHUD(); animate();
      flashStatus('Press Space to fire');
    }
    start();

    // ---------- Resize ----------
    addEventListener('resize', ()=>{
      camera.aspect = innerWidth/innerHeight; camera.updateProjectionMatrix();
      renderer.setSize(innerWidth, innerHeight);
    });
  </script>
</body>
