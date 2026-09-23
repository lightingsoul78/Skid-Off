# Skid-Off
Drifting game 
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Drift Escape</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body {
    background: #0a0a14;
    color: #fff;
    font-family: 'Courier New', monospace;
    overflow: hidden;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
  }
  canvas {
    background: #1a1a2e;
    border: 2px solid #ff006e;
    box-shadow: 0 0 40px rgba(255, 0, 110, 0.5);
  }
  #overlay {
    position: absolute;
    text-align: center;
    pointer-events: none;
    color: #fff;
    text-shadow: 0 0 10px #ff006e;
  }
  #overlay h1 { font-size: 48px; letter-spacing: 4px; }
  #overlay p { font-size: 18px; margin-top: 10px; opacity: 0.9; }
  .blink { animation: blink 1s infinite; }
  @keyframes blink { 50% { opacity: 0.3; } }
</style>
</head>
<body>
<canvas id="game" width="900" height="600"></canvas>
<div id="overlay">
  <h1>DRIFT ESCAPE</h1>
  <p>Arrow Keys / WASD to drive · SPACE to handbrake drift</p>
  <p>Collect green checkpoints · Escape the police!</p>
  <p class="blink" style="margin-top:30px">Press ENTER to start</p>
</div>

<script>
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const overlay = document.getElementById('overlay');
const W = canvas.width, H = canvas.height;

// Game state
let state = 'menu'; // menu, playing, gameover
let keys = {};
let score = 0;
let timeLeft = 60;
let driftCombo = 0;
let driftMultiplier = 1;
let lastTime = performance.now();

// World
const WORLD = { w: 3000, h: 3000 };
let camera = { x: 0, y: 0 };

// Car (player)
const car = {
  x: WORLD.w / 2, y: WORLD.h / 2,
  vx: 0, vy: 0,
  angle: -Math.PI / 2,
  width: 22, height: 40,
  speed: 0,
  driftAngle: 0,
  isDrifting: false
};

// Police
const police = [];
function spawnPolice() {
  const angle = Math.random() * Math.PI * 2;
  const dist = 600 + Math.random() * 200;
  police.push({
    x: car.x + Math.cos(angle) * dist,
    y: car.y + Math.sin(angle) * dist,
    vx: 0, vy: 0,
    angle: 0,
    speed: 0,
    width: 22, height: 40,
    siren: 0
  });
}

// Checkpoints
const checkpoints = [];
function spawnCheckpoint() {
  const angle = Math.random() * Math.PI * 2;
  const dist = 400 + Math.random() * 600;
  checkpoints.push({
    x: car.x + Math.cos(angle) * dist,
    y: car.y + Math.sin(angle) * dist,
    radius: 25,
    pulse: 0
  });
}

// Tire marks
const skidMarks = [];

// Smoke particles
const smoke = [];

// Input
window.addEventListener('keydown', e => {
  keys[e.key.toLowerCase()] = true;
  if (e.key === 'Enter') {
    if (state === 'menu' || state === 'gameover') startGame();
  }
});
window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

function startGame() {
  state = 'playing';
  overlay.style.display = 'none';
  score = 0;
  timeLeft = 60;
  driftCombo = 0;
  driftMultiplier = 1;
  car.x = WORLD.w / 2;
  car.y = WORLD.h / 2;
  car.vx = 0; car.vy = 0;
  car.angle = -Math.PI / 2;
  police.length = 0;
  checkpoints.length = 0;
  skidMarks.length = 0;
  smoke.length = 0;
  for (let i = 0; i < 2; i++) spawnPolice();
  for (let i = 0; i < 4; i++) spawnCheckpoint();
}

function endGame() {
  state = 'gameover';
  overlay.style.display = 'block';
  overlay.innerHTML = `
    <h1>BUSTED!</h1>
    <p>Final Score: ${Math.floor(score)}</p>
    <p>Drift Chains Survived: ${Math.floor(driftCombo)}</p>
    <p class="blink" style="margin-top:30px">Press ENTER to retry</p>
  `;
}

// Physics update
function updateCar(dt) {
  const accel = keys['arrowup'] || keys['w'] ? 300 : 0;
  const brake = keys['arrowdown'] || keys['s'] ? 400 : 0;
  const steerL = keys['arrowleft'] || keys['a'] ? 1 : 0;
  const steerR = keys['arrowright'] || keys['d'] ? 1 : 0;
  const handbrake = keys[' '] || keys['spacebar'];

  // Forward direction
  const fx = Math.cos(car.angle);
  const fy = Math.sin(car.angle);

  // Current speed along heading
  const forwardSpeed = car.vx * fx + car.vy * fy;

  // Steering (only when moving)
  const steerAmount = (steerR - steerL) * 2.8 * Math.min(1, Math.abs(forwardSpeed) / 100);
  car.angle += steerAmount * dt;

  // Acceleration along heading
  const newFx = Math.cos(car.angle);
  const newFy = Math.sin(car.angle);
  car.vx += newFx * accel * dt;
  car.vy += newFy * accel * dt;

  // Braking
  if (brake > 0) {
    const sp = Math.hypot(car.vx, car.vy);
    if (sp > 0) {
      const decel = Math.min(sp, brake * dt);
      car.vx -= (car.vx / sp) * decel;
      car.vy -= (car.vy / sp) * decel;
    }
  }

  // Grip: split velocity into forward/lateral
  const forward = car.vx * newFx + car.vy * newFy;
  const lateral = car.vx * (-newFy) + car.vy * newFx;

  // Grip factor (handbrake reduces grip heavily)
  const grip = handbrake ? 0.85 : 0.98;
  const newLateral = lateral * Math.pow(grip, dt * 60);

  // Reconstruct velocity
  car.vx = newFx * forward + (-newFy) * newLateral;
  car.vy = newFy * forward + newFx * newLateral;

  // Rolling friction
  const friction = handbrake ? 0.995 : 0.99;
  car.vx *= Math.pow(friction, dt * 60);
  car.vy *= Math.pow(friction, dt * 60);

  // Position
  car.x += car.vx * dt;
  car.y += car.vy * dt;

  // World bounds
  car.x = Math.max(50, Math.min(WORLD.w - 50, car.x));
  car.y = Math.max(50, Math.min(WORLD.h - 50, car.y));

  // Drift detection
  car.speed = Math.hypot(car.vx, car.vy);
  const velAngle = Math.atan2(car.vy, car.vx);
  car.driftAngle = Math.abs(angleDiff(car.angle, velAngle));
  car.isDrifting = car.driftAngle > 0.3 && car.speed > 100;

  // Skid marks when drifting or handbraking at speed
  if ((car.isDrifting || (handbrake && car.speed > 80)) && car.speed > 50) {
    const backX = car.x - newFx * 15;
    const backY = car.y - newFy * 15;
    const perpX = -newFy * 8;
    const perpY = newFx * 8;
    skidMarks.push({ x: backX + perpX, y: backY + perpY, life: 1 });
    skidMarks.push({ x: backX - perpX, y: backY - perpY, life: 1 });
    // Smoke
    if (Math.random() < 0.4) {
      smoke.push({
        x: backX + (Math.random() - 0.5) * 10,
        y: backY + (Math.random() - 0.5) * 10,
        vx: (Math.random() - 0.5) * 20,
        vy: (Math.random() - 0.5) * 20,
        life: 1,
        size: 8 + Math.random() * 8
      });
    }
  }

  // Drift scoring
  if (car.isDrifting) {
    const driftScore = car.driftAngle * car.speed * dt * 0.1 * driftMultiplier;
    score += driftScore;
    driftCombo += dt;
    driftMultiplier = 1 + Math.floor(driftCombo);
  } else {
    driftCombo = 0;
    driftMultiplier = 1;
  }

  // Fade skid marks
  for (let i = skidMarks.length - 1; i >= 0; i--) {
    skidMarks[i].life -= dt * 0.15;
    if (skidMarks[i].life <= 0) skidMarks.splice(i, 1);
  }
  if (skidMarks.length > 1500) skidMarks.splice(0, skidMarks.length - 1500);

  // Update smoke
  for (let i = smoke.length - 1; i >= 0; i--) {
    const p = smoke[i];
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.life -= dt * 1.5;
    p.size += dt * 20;
    if (p.life <= 0) smoke.splice(i, 1);
  }
}

function angleDiff(a, b) {
  let d = b - a;
  while (d > Math.PI) d -= Math.PI * 2;
  while (d < -Math.PI) d += Math.PI * 2;
  return d;
}

function updatePolice(dt) {
  for (const p of police) {
    const dx = car.x - p.x;
    const dy = car.y - p.y;
    const dist = Math.hypot(dx, dy);
    const targetAngle = Math.atan2(dy, dx);
    p.angle += angleDiff(p.angle, targetAngle) * Math.min(1, dt * 3);

    const fx = Math.cos(p.angle);
    const fy = Math.sin(p.angle);
    const targetSpeed = Math.min(260, dist * 0.8 + 80);
    p.speed += (targetSpeed - p.speed) * dt * 2;
    p.vx = fx * p.speed;
    p.vy = fy * p.speed;
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.siren = (p.siren + dt * 8) % (Math.PI * 2);

    // Collision with player
    if (dist < 30) {
      endGame();
      return;
    }
  }
}

function updateCheckpoints(dt) {
  for (let i = checkpoints.length - 1; i >= 0; i--) {
    const c = checkpoints[i];
    c.pulse += dt * 4;
    const dx = car.x - c.x;
    const dy = car.y - c.y;
    if (Math.hypot(dx, dy) < c.radius + 15) {
      checkpoints.splice(i, 1);
      score += 500 * driftMultiplier;
      timeLeft += 8;
      spawnCheckpoint();
      // Spawn more police occasionally
      if (Math.random() < 0.5 && police.length < 6) spawnPolice();
    }
  }
}

// Rendering
function draw() {
  // Camera follows car
  camera.x = car.x - W / 2;
  camera.y = car.y - H / 2;

  ctx.fillStyle = '#1a1a2e';
  ctx.fillRect(0, 0, W, H);

  ctx.save();
  ctx.translate(-camera.x, -camera.y);

  // Grid background
  ctx.strokeStyle = '#252545';
  ctx.lineWidth = 1;
  const gridSize = 80;
  const startX = Math.floor(camera.x / gridSize) * gridSize;
  const startY = Math.floor(camera.y / gridSize) * gridSize;
  for (let x = startX; x < camera.x + W + gridSize; x += gridSize) {
    ctx.beginPath();
    ctx.moveTo(x, camera.y);
    ctx.lineTo(x, camera.y + H);
    ctx.stroke();
  }
  for (let y = startY; y < camera.y + H + gridSize; y += gridSize) {
    ctx.beginPath();
    ctx.moveTo(camera.x, y);
    ctx.lineTo(camera.x + W, y);
    ctx.stroke();
  }

  // World border
  ctx.strokeStyle = '#ff006e';
  ctx.lineWidth = 4;
  ctx.strokeRect(0, 0, WORLD.w, WORLD.h);

  // Skid marks
  for (const s of skidMarks) {
    ctx.fillStyle = `rgba(20, 20, 20, ${s.life * 0.7})`;
    ctx.fillRect(s.x - 2, s.y - 2, 4, 4);
  }

  // Smoke
  for (const p of smoke) {
    ctx.fillStyle = `rgba(200, 200, 220, ${p.life * 0.4})`;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
    ctx.fill();
  }

  // Checkpoints
  for (const c of checkpoints) {
    const pulse = 1 + Math.sin(c.pulse) * 0.2;
    ctx.fillStyle = 'rgba(0, 255, 150, 0.2)';
    ctx.beginPath();
    ctx.arc(c.x, c.y, c.radius * pulse * 1.5, 0, Math.PI * 2);
    ctx.fill();
    ctx.strokeStyle = '#00ff96';
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.arc(c.x, c.y, c.radius * pulse, 0, Math.PI * 2);
    ctx.stroke();
    ctx.fillStyle = '#00ff96';
    ctx.font = 'bold 20px Courier New';
    ctx.textAlign = 'center';
    ctx.fillText('+', c.x, c.y + 7);
  }

  // Police
  for (const p of police) {
    drawCar(p, '#ff006e', Math.sin(p.siren) > 0 ? '#ff006e' : '#00aaff');
  }

  // Player
  drawCar(car, '#00d9ff', '#ffeb3b');

  ctx.restore();

  // HUD
  drawHUD();
}

function drawCar(c, bodyColor, lightColor) {
  ctx.save();
  ctx.translate(c.x, c.y);
  ctx.rotate(c.angle);

  // Shadow
  ctx.fillStyle = 'rgba(0,0,0,0.4)';
  ctx.fillRect(-c.width/2 + 2, -c.height/2 + 2, c.width, c.height);

  // Body
  ctx.fillStyle = bodyColor;
  ctx.fillRect(-c.width/2, -c.height/2, c.width, c.height);

  // Windshield
  ctx.fillStyle = 'rgba(20, 30, 50, 0.8)';
  ctx.fillRect(-c.width/2 + 3, -c.height/2 + 6, c.width - 6, 10);

  // Rear
  ctx.fillStyle = 'rgba(0,0,0,0.4)';
  ctx.fillRect(-c.width/2 + 3, c.height/2 - 8, c.width - 6, 5);

  // Headlights
  ctx.fillStyle = lightColor;
  ctx.fillRect(-c.width/2 + 2, -c.height/2, 4, 3);
  ctx.fillRect(c.width/2 - 6, -c.height/2, 4, 3);

  // Outline
  ctx.strokeStyle = 'rgba(255,255,255,0.3)';
  ctx.lineWidth = 1;
  ctx.strokeRect(-c.width/2, -c.height/2, c.width, c.height);

  ctx.restore();
}

function drawHUD() {
  // Top bar
  ctx.fillStyle = 'rgba(10, 10, 20, 0.7)';
  ctx.fillRect(0, 0, W, 50);

  ctx.fillStyle = '#00d9ff';
  ctx.font = 'bold 20px Courier New';
  ctx.textAlign = 'left';
  ctx.fillText(`SCORE: ${Math.floor(score)}`, 20, 32);

  // Time
  ctx.fillStyle = timeLeft < 10 ? '#ff006e' : '#ffeb3b';
  ctx.textAlign = 'center';
  ctx.fillText(`TIME: ${Math.ceil(timeLeft)}s`, W/2, 32);

  // Multiplier
  if (driftMultiplier > 1) {
    ctx.fillStyle = '#ff006e';
    ctx.textAlign = 'right';
    ctx.fillText(`x${driftMultiplier} DRIFT`, W - 20, 32);
  }

  // Speed
  ctx.fillStyle = '#fff';
  ctx.textAlign = 'left';
  ctx.font = '14px Courier New';
  ctx.fillText(`${Math.floor(car.speed * 0.5)} km/h`, 20, H - 20);

  // Drift indicator
  if (car.isDrifting) {
    ctx.fillStyle = `rgba(255, 0, 110, ${0.5 + Math.sin(Date.now() / 100) * 0.3})`;
    ctx.font = 'bold 36px Courier New';
    ctx.textAlign = 'center';
    ctx.fillText('DRIFTING!', W/2, H - 40);
  }

  // Minimap
  const mmSize = 120;
  const mmX = W - mmSize - 15;
  const mmY = H - mmSize - 15;
  ctx.fillStyle = 'rgba(10, 10, 20, 0.7)';
  ctx.fillRect(mmX, mmY, mmSize, mmSize);
  ctx.strokeStyle = '#ff006e';
  ctx.lineWidth = 1;
  ctx.strokeRect(mmX, mmY, mmSize, mmSize);

  const scaleX = mmSize / WORLD.w;
  const scaleY = mmSize / WORLD.h;

  // Checkpoints on minimap
  ctx.fillStyle = '#00ff96';
  for (const c of checkpoints) {
    ctx.fillRect(mmX + c.x * scaleX - 2, mmY + c.y * scaleY - 2, 4, 4);
  }
  // Police on minimap
  ctx.fillStyle = '#ff006e';
  for (const p of police) {
    ctx.fillRect(mmX + p.x * scaleX - 2, mmY + p.y * scaleY - 2, 4, 4);
  }
  // Player on minimap
  ctx.fillStyle = '#00d9ff';
  ctx.fillRect(mmX + car.x * scaleX - 3, mmY + car.y * scaleY - 3, 6, 6);
}

// Main loop
function loop(now) {
  const dt = Math.min(0.05, (now - lastTime) / 1000);
  lastTime = now;

  if (state === 'playing') {
    timeLeft -= dt;
    if (timeLeft <= 0) {
      timeLeft = 0;
      endGame();
    } else {
      updateCar(dt);
      updatePolice(dt);
      updateCheckpoints(dt);
    }
  }

  draw();
  requestAnimationFrame(loop);
}

requestAnimationFrame(loop);
</script>
</body>
</html>
