<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>FPS 3D - Demo</title>
  <style>
    html,body{height:100%;margin:0;font-family:Inter,Arial,Helvetica,sans-serif;overflow:hidden;background:#0b0f14}
    #hud{position:fixed;left:12px;top:12px;color:#fff;z-index:30;background:rgba(0,0,0,0.35);padding:10px;border-radius:8px}
    #cross{position:fixed;left:50%;top:50%;transform:translate(-50%,-50%);width:18px;height:18px;z-index:29;pointer-events:none}
    #cross:before,#cross:after{content:'';position:absolute;background:#fff}
    #cross:before{left:8px;top:0;bottom:0;width:2px}
    #cross:after{top:8px;left:0;right:0;height:2px}
    #start{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;z-index:40;background:linear-gradient(180deg, rgba(3,6,10,0.7), rgba(0,0,0,0.85));color:#fff;flex-direction:column}
    button{background:#1e90ff;border:none;padding:10px 14px;border-radius:8px;color:#fff;cursor:pointer;font-weight:600}
    #weaponCanvas{position:fixed;right:0;bottom:0;pointer-events:none}
    #msg{position:fixed;bottom:14px;left:50%;transform:translateX(-50%);color:#fff;z-index:30;opacity:0.85}
    .small{opacity:0.85;font-size:13px;margin-top:6px}
    @media (max-width:700px){#hud{left:8px;right:8px;top:auto;bottom:12px}}
  </style>
</head>
<body>
  <div id="hud">Health: <span id="hp">100</span> · Score: <span id="score">0</span></div>
  <div id="cross"></div>
  <div id="msg">W/A/S/D - Move · Click - Shoot · Space - Jump · R - Respawn</div>
  <div id="start">
    <h1 style="margin:0 0 8px 0">FPS 3D (Demo)</h1>
    <p style="margin:0 0 12px 0;opacity:0.9">Click Start then click inside the canvas to lock the mouse (Pointer Lock).</p>
    <button id="startBtn">Start</button>
    <div class="small">Pistol + visible weapon model · Enemies move & shoot · Works in modern browsers</div>
  </div>

  <script src="https://unpkg.com/three@0.152.2/build/three.min.js"></script>
  <script>
  // FPS demo using Three.js
  // Save as fps.html and open in Chrome/Firefox. Single-file.

  // Scene setup
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x87ceeb);
  const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 1000);

  const renderer = new THREE.WebGLRenderer({antialias:true});
  renderer.setSize(innerWidth, innerHeight); renderer.setPixelRatio(Math.min(devicePixelRatio,2));
  document.body.appendChild(renderer.domElement);

  // Light
  const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 0.9); scene.add(hemi);
  const dir = new THREE.DirectionalLight(0xffffff, 0.6); dir.position.set(5,10,2); scene.add(dir);

  // Ground
  const groundMat = new THREE.MeshStandardMaterial({color:0x556b2f, roughness:1});
  const ground = new THREE.Mesh(new THREE.PlaneGeometry(500,500), groundMat);
  ground.rotation.x = -Math.PI/2; ground.position.y = 0; scene.add(ground);

  // Player container
  const player = new THREE.Object3D(); player.position.set(0,1.6,6); scene.add(player); player.add(camera);

  // Add a simple visible pistol attached to camera
  const weapon = new THREE.Group();
  const barrel = new THREE.Mesh(new THREE.BoxGeometry(0.06,0.06,0.5), new THREE.MeshStandardMaterial({color:0x222222, metalness:0.6, roughness:0.4}));
  barrel.position.set(0,-0.05,-0.5);
  const grip = new THREE.Mesh(new THREE.BoxGeometry(0.12,0.08,0.2), new THREE.MeshStandardMaterial({color:0x1b1b1b}));
  grip.position.set(0.12,-0.12,-0.15);
  weapon.add(barrel, grip);
  weapon.position.set(0.3,-0.25,-0.6);
  camera.add(weapon);

  // Targets / enemies
  const enemies = new THREE.Group(); scene.add(enemies);
  const enemyGeo = new THREE.BoxGeometry(0.8,1.6,0.6);
  const enemyMat = new THREE.MeshStandardMaterial({color:0x8b0000});
  function spawnEnemy(x,z){
    const m = new THREE.Mesh(enemyGeo, enemyMat.clone());
    m.position.set(x,0.8,z); m.userData.hp = 3; m.userData.reload = 0; enemies.add(m);
  }
  for(let i=0;i<6;i++) spawnEnemy((Math.random()-0.5)*40, -10 - Math.random()*60);

  // Bullets (visual tracers) array
  const bullets = [];

  // HUD refs
  const hpEl = document.getElementById('hp'); const scoreEl = document.getElementById('score');
  let health = 100; let score = 0;

  // Controls
  const move = {f:0,b:0,l:0,r:0}; let canJump = false; const velocity = new THREE.Vector3();
  window.addEventListener('keydown', (e)=>{
    if(e.code==='KeyW') move.f=1; if(e.code==='KeyS') move.b=1; if(e.code==='KeyA') move.l=1; if(e.code==='KeyD') move.r=1;
    if(e.code==='Space' && canJump){ velocity.y = 6; canJump=false; }
    if(e.code==='KeyR') respawn();
  });
  window.addEventListener('keyup', (e)=>{ if(e.code==='KeyW') move.f=0; if(e.code==='KeyS') move.b=0; if(e.code==='KeyA') move.l=0; if(e.code==='KeyD') move.r=0; });

  // Mouse look
  let pitch=0,yaw=0;
  function onMouseMove(e){ if(document.pointerLockElement===renderer.domElement){ yaw -= e.movementX*0.002; pitch -= e.movementY*0.002; pitch = Math.max(-Math.PI/2+0.05, Math.min(Math.PI/2-0.05, pitch)); player.rotation.y = yaw; camera.rotation.x = pitch; } }
  window.addEventListener('mousemove', onMouseMove);

  // Shooting logic
  let canShoot = true; const fireRate = 0.18; let lastShot = 0;
  renderer.domElement.addEventListener('mousedown', ()=>{ if(document.pointerLockElement===renderer.domElement) tryShoot(); });

  function tryShoot(){ const now = performance.now()/1000; if(now - lastShot < fireRate) return; lastShot = now; shoot(); }
  function shoot(){
    // Recoil animation
    weapon.position.y -= 0.04; weapon.position.z += 0.04; setTimeout(()=>{ weapon.position.y += 0.04; weapon.position.z -= 0.04; }, 80);
    // Raycast
    const origin = new THREE.Vector3(); origin.copy(camera.getWorldPosition(new THREE.Vector3()));
    const dir = new THREE.Vector3(0,0,-1).applyQuaternion(camera.quaternion);
    const ray = new THREE.Raycaster(origin, dir, 0, 200);
    const hit = ray.intersectObjects(enemies.children, false);
    if(hit.length>0){ const t = hit[0].object; t.userData.hp -= 1; if(t.userData.hp<=0){ enemies.remove(t); score += 25; scoreEl.textContent = score; setTimeout(()=>spawnEnemy((Math.random()-0.5)*40, -10 - Math.random()*60), 1000); } }
    // muzzle flash
    const flash = new THREE.Mesh(new THREE.SphereGeometry(0.06,6,6), new THREE.MeshBasicMaterial({color:0xffdd88})); flash.position.copy(origin.clone().add(dir.clone().multiplyScalar(0.6))); scene.add(flash); setTimeout(()=>scene.remove(flash),50);
    // tracer
    bullets.push({pos: origin.clone(), dir: dir.clone(), life: 0.5});
  }

  // Enemy shooting - when close enough
  function enemyAI(dt){ enemies.children.forEach((e)=>{
    const toPlayer = player.position.clone().sub(e.position); const d = toPlayer.length(); if(d>0.6){ e.position.add(toPlayer.normalize().multiplyScalar(1.0*dt)); }
    // rotate to face
    const angle = Math.atan2(player.position.x - e.position.x, player.position.z - e.position.z);
    e.rotation.y = angle;
    // shoot if in range
    e.userData.reload = Math.max(0, e.userData.reload - dt);
    if(d < 12 && e.userData.reload <= 0){ e.userData.reload = 2.0; // shoot
      // if facing roughly the player, apply damage
      const dirTo = player.position.clone().sub(e.position).normalize(); const aim = dirTo.dot(new THREE.Vector3(0,0,-1).applyQuaternion(new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,1,0), e.rotation.y)));
      if(aim > 0.4){ health -= 12; hpEl.textContent = Math.max(0, Math.floor(health)); if(health<=0) onDeath(); }
    }
  }); }

  function onDeath(){ document.getElementById('start').style.display='flex'; document.getElementById('start').querySelector('h1').textContent='You Died'; document.getElementById('start').querySelector('p').textContent='Score: '+score+' · Press Start to replay.'; }

  // Respawn
  function respawn(){ health=100; score=0; hpEl.textContent=health; scoreEl.textContent=score; player.position.set(0,1.6,6); while(enemies.children.length) enemies.remove(enemies.children[0]); for(let i=0;i<6;i++) spawnEnemy((Math.random()-0.5)*40, -10 - Math.random()*60); document.getElementById('start').style.display='none'; }

  // Animation loop
  const clock = new THREE.Clock();
  function animate(){ const dt = Math.min(0.05, clock.getDelta());
    // movement
    const speed = 6.0; const dir = new THREE.Vector3(); if(move.f) dir.z -= 1; if(move.b) dir.z += 1; if(move.l) dir.x -= 1; if(move.r) dir.x += 1; if(dir.lengthSq()>0){ dir.normalize(); const forward = dir.clone().applyAxisAngle(new THREE.Vector3(0,1,0), player.rotation.y); player.position.add(forward.multiplyScalar(speed*dt)); }
    // gravity
    velocity.y -= 9.8*dt; player.position.y += velocity.y*dt; if(player.position.y <= 1.6){ player.position.y = 1.6; velocity.y = 0; canJump = true; }

    // bullets update
    for(let i=bullets.length-1;i>=0;i--){ const b = bullets[i]; b.life -= dt; if(b.life<=0){ bullets.splice(i,1); continue; } b.pos.add(b.dir.clone().multiplyScalar(120*dt)); // draw tracer
      // simple visual tracer - small sphere
      if(!b.mesh){ const m = new THREE.Mesh(new THREE.SphereGeometry(0.03,6,6), new THREE.MeshBasicMaterial()); m.material.color.setHex(0xffffcc); m.position.copy(b.pos); scene.add(m); b.mesh = m; } else { b.mesh.position.copy(b.pos); }
      // check collision with enemies
      for(const en of enemies.children){ if(en.position.distanceTo(b.pos) < 0.8){ en.userData.hp -= 1; if(en.userData.hp<=0){ enemies.remove(en); score+=25; scoreEl.textContent=score; setTimeout(()=>spawnEnemy((Math.random()-0.5)*40, -10 - Math.random()*60), 1000); } if(b.mesh){ scene.remove(b.mesh); } bullets.splice(i,1); break; } }
    }

    // remove tracer meshes when life ended
    bullets.forEach(b=>{ if(b.life<=0 && b.mesh){ scene.remove(b.mesh); } });

    // enemy AI
    enemyAI(dt);

    renderer.render(scene, camera);
    requestAnimationFrame(animate);
  }

  // Start button + pointer lock
  const startBtn = document.getElementById('startBtn'); startBtn.addEventListener('click', ()=>{ document.getElementById('start').style.display='none'; renderer.domElement.requestPointerLock(); if(!animate._started){ animate._started = true; animate(); } });
  document.addEventListener('pointerlockchange', ()=>{ if(document.pointerLockElement === renderer.domElement){ document.getElementById('cross').style.display='block'; } else { document.getElementById('cross').style.display='none'; } });

  // Resize
  window.addEventListener('resize', ()=>{ camera.aspect = innerWidth/innerHeight; camera.updateProjectionMatrix(); renderer.setSize(innerWidth, innerHeight); });

  // Prevent context menu
  window.addEventListener('contextmenu', e=>e.preventDefault());

  </script>
</body>
</html>
