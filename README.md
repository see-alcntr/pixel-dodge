const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const waveText = document.getElementById('waveText');
const scoreText = document.getElementById('scoreText');
const hpText = document.getElementById('hpText');
const weaponText = document.getElementById('weaponText');
const ammoText = document.getElementById('ammoText');
const objectiveText = document.getElementById('objectiveText');
const overlay = document.getElementById('overlay');
const startButton = document.getElementById('startButton');

const world = {
  width: canvas.width,
  height: canvas.height,
};

let input = {
  keys: {},
  mouseX: canvas.width / 2,
  mouseY: canvas.height / 2,
  isMouseDown: false,
};

const game = {
  running: false,
  over: false,
  score: 0,
  wave: 1,
  enemyCap: 5,
  enemyTimer: 0,
  lastTime: 0,
  shake: 0,
};

const player = {
  x: world.width / 2,
  y: world.height / 2,
  radius: 16,
  speed: 220,
  sprintSpeed: 320,
  angle: 0,
  hp: 100,
  maxHp: 100,
  score: 0,
  ammo: 30,
  reserveAmmo: 120,
  magSize: 30,
  reloadTime: 1.3,
  reloadTimer: 0,
  fireDelay: 0.12,
  fireTimer: 0,
  flash: 0,
};

const bullets = [];
const enemies = [];
const particles = [];
const walls = [
  { x: 180, y: 110, w: 120, h: 18 },
  { x: 660, y: 110, w: 120, h: 18 },
  { x: 180, y: 410, w: 120, h: 18 },
  { x: 660, y: 410, w: 120, h: 18 },
  { x: 470, y: 210, w: 18, h: 120 },
  { x: 460, y: 330, w: 18, h: 90 },
  { x: 450, y: 100, w: 18, h: 90 },
];

function resetGame() {
  player.x = world.width / 2;
  player.y = world.height / 2;
  player.hp = player.maxHp;
  player.ammo = player.magSize;
  player.reserveAmmo = 120;
  player.reloadTimer = 0;
  player.fireTimer = 0;
  player.flash = 0;
  game.score = 0;
  game.wave = 1;
  game.enemyCap = 5;
  game.enemyTimer = 0;
  game.shake = 0;
  bullets.length = 0;
  enemies.length = 0;
  particles.length = 0;
  updateHud();
}

function startGame() {
  resetGame();
  game.running = true;
  game.over = false;
  overlay.classList.remove('visible');
}

function endGame() {
  game.running = false;
  game.over = true;
  objectiveText.textContent = 'Mission failed';
  overlay.classList.add('visible');
  overlay.querySelector('h1').textContent = 'Mission Failed';
  overlay.querySelector('p').textContent = 'You were taken out. Press deploy to redeploy.';
  overlay.querySelector('ul').innerHTML = `
    <li>Final score: ${game.score}</li>
    <li>Reached wave: ${game.wave}</li>
    <li>Try again and improve your line.</li>
  `;
}

function updateHud() {
  waveText.textContent = String(game.wave);
  scoreText.textContent = String(game.score);
  hpText.textContent = String(Math.max(0, Math.ceil(player.hp)));
  weaponText.textContent = 'Ranger M4';
  ammoText.textContent = `${player.ammo} / ${player.reserveAmmo}`;
  objectiveText.textContent = `Wave ${game.wave} — clear the zone`;
}

function clamp(value, min, max) {
  return Math.min(max, Math.max(min, value));
}

function rectCircleCollision(circle, rect) {
  const closestX = clamp(circle.x, rect.x, rect.x + rect.w);
  const closestY = clamp(circle.y, rect.y, rect.y + rect.h);
  const dx = circle.x - closestX;
  const dy = circle.y - closestY;
  return dx * dx + dy * dy < circle.r * circle.r;
}

function movePlayer(dx, dy) {
  const speed = input.keys['Shift'] ? player.sprintSpeed : player.speed;
  const nextX = player.x + dx * speed;
  const nextY = player.y + dy * speed;

  const candidate = { x: nextX, y: player.y, r: player.radius };
  const hitX = walls.some((wall) => rectCircleCollision(candidate, wall));
  if (!hitX) player.x = clamp(nextX, player.radius, world.width - player.radius);

  const candidateY = { x: player.x, y: nextY, r: player.radius };
  const hitY = walls.some((wall) => rectCircleCollision(candidateY, wall));
  if (!hitY) player.y = clamp(nextY, player.radius, world.height - player.radius);
}

function shoot() {
  if (!game.running || player.reloadTimer > 0) return;
  if (player.ammo <= 0) {
    reload();
    return;
  }

  if (player.fireTimer > 0) return;

  const angle = Math.atan2(input.mouseY - player.y, input.mouseX - player.x);
  const speed = 520;
  bullets.push({
    x: player.x + Math.cos(angle) * (player.radius + 8),
    y: player.y + Math.sin(angle) * (player.radius + 8),
    vx: Math.cos(angle) * speed,
    vy: Math.sin(angle) * speed,
    r: 4,
    life: 1.2,
    damage: 24,
  });

  player.ammo -= 1;
  player.fireTimer = player.fireDelay;
  player.flash = 0.08;
  updateHud();
}

function reload() {
  if (player.reloadTimer > 0 || player.ammo === player.magSize || player.reserveAmmo <= 0) return;
  player.reloadTimer = player.reloadTime;
}

function handleReload(dt) {
  if (player.reloadTimer > 0) {
    player.reloadTimer -= dt;
    if (player.reloadTimer <= 0) {
      const needed = player.magSize - player.ammo;
      const loaded = Math.min(needed, player.reserveAmmo);
      player.ammo += loaded;
      player.reserveAmmo -= loaded;
      updateHud();
    }
  }
}

function spawnEnemy() {
  const side = Math.floor(Math.random() * 4);
  let x = 0;
  let y = 0;

  if (side === 0) {
    x = Math.random() * world.width;
    y = -20;
  } else if (side === 1) {
    x = world.width + 20;
    y = Math.random() * world.height;
  } else if (side === 2) {
    x = Math.random() * world.width;
    y = world.height + 20;
  } else {
    x = -20;
    y = Math.random() * world.height;
  }

  const typeRoll = Math.random();
  const type = typeRoll < 0.7 ? 'grunt' : 'runner';

  enemies.push({
    x,
    y,
    r: type === 'grunt' ? 16 : 12,
    hp: type === 'grunt' ? 45 : 24,
    maxHp: type === 'grunt' ? 45 : 24,
    speed: type === 'grunt' ? 78 : 102,
    damage: type === 'grunt' ? 12 : 9,
    color: type === 'grunt' ? '#ff6a6a' : '#ffca70',
    cooldown: 0.9,
  });
}

function spawnParticles(x, y, color, count = 10) {
  for (let i = 0; i < count; i += 1) {
    particles.push({
      x,
      y,
      vx: (Math.random() - 0.5) * 140,
      vy: (Math.random() - 0.5) * 140,
      life: 0.5 + Math.random() * 0.5,
      size: 2 + Math.random() * 4,
      color,
    });
  }
}

function updateBullets(dt) {
  for (let i = bullets.length - 1; i >= 0; i -= 1) {
    const bullet = bullets[i];
    bullet.x += bullet.vx * dt;
    bullet.y += bullet.vy * dt;
    bullet.life -= dt;

    if (
      bullet.x < -10 ||
      bullet.x > world.width + 10 ||
      bullet.y < -10 ||
      bullet.y > world.height + 10 ||
      bullet.life <= 0
    ) {
      bullets.splice(i, 1);
      continue;
    }

    let hit = false;
    for (let j = enemies.length - 1; j >= 0; j -= 1) {
      const enemy = enemies[j];
      const dx = bullet.x - enemy.x;
      const dy = bullet.y - enemy.y;
      if (dx * dx + dy * dy <= (bullet.r + enemy.r) ** 2) {
        enemy.hp -= bullet.damage;
        hit = true;
        spawnParticles(bullet.x, bullet.y, '#9de7ff', 8);

        if (enemy.hp <= 0) {
          game.score += 10;
          spawnParticles(enemy.x, enemy.y, enemy.color, 20);
          enemies.splice(j, 1);
          game.shake = 8;
        }
        break;
      }
    }

    if (hit) {
      bullets.splice(i, 1);
    }
  }
}

function updateEnemies(dt) {
  for (let i = enemies.length - 1; i >= 0; i -= 1) {
    const enemy = enemies[i];
    const angle = Math.atan2(player.y - enemy.y, player.x - enemy.x);
    enemy.x += Math.cos(angle) * enemy.speed * dt;
    enemy.y += Math.sin(angle) * enemy.speed * dt;

    const candidate = { x: enemy.x, y: enemy.y, r: enemy.r };
    let blocked = false;
    for (const wall of walls) {
      if (rectCircleCollision(candidate, wall)) {
        blocked = true;
        break;
      }
    }

    if (blocked) {
      enemy.x -= Math.cos(angle) * enemy.speed * dt * 0.8;
      enemy.y -= Math.sin(angle) * enemy.speed * dt * 0.8;
    }

    const dist = Math.hypot(player.x - enemy.x, player.y - enemy.y);
    if (dist < enemy.r + player.radius + 6) {
      enemy.cooldown -= dt;
      if (enemy.cooldown <= 0) {
        player.hp -= enemy.damage;
        enemy.cooldown = 0.9;
        game.shake = 10;
        spawnParticles(player.x, player.y, '#ff7d7d', 12);
      }
    } else {
      enemy.cooldown = Math.max(0.2, enemy.cooldown - dt);
    }

    if (player.hp <= 0) {
      endGame();
      return;
    }
  }
}

function updateParticles(dt) {
  for (let i = particles.length - 1; i >= 0; i -= 1) {
    const p = particles[i];
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.life -= dt;

    if (p.life <= 0) particles.splice(i, 1);
  }
}

function updateWave() {
  if (enemies.length === 0 && game.running) {
    game.wave += 1;
    game.enemyCap += 3;
    const spawnCount = Math.min(3 + game.wave, game.enemyCap);
    for (let i = 0; i < spawnCount; i += 1) {
      spawnEnemy();
    }
  }
}

function update(dt) {
  if (!game.running) return;

  const dx = (input.keys['KeyD'] ? 1 : 0) - (input.keys['KeyA'] ? 1 : 0);
  const dy = (input.keys['KeyS'] ? 1 : 0) - (input.keys['KeyW'] ? 1 : 0);
  if (dx !== 0 || dy !== 0) {
    const len = Math.hypot(dx, dy) || 1;
    movePlayer(dx / len, dy / len);
  }

  player.fireTimer = Math.max(0, player.fireTimer - dt);
  player.flash = Math.max(0, player.flash - dt);
  handleReload(dt);

  if (input.isMouseDown) shoot();
  if (game.enemyTimer <= 0 && enemies.length < game.enemyCap) {
    spawnEnemy();
    game.enemyTimer = Math.max(0.35, 1.2 - game.wave * 0.06);
  }
  game.enemyTimer -= dt;

  updateBullets(dt);
  updateEnemies(dt);
  updateParticles(dt);
  updateWave();

  player.angle = Math.atan2(input.mouseY - player.y, input.mouseX - player.x);
  game.shake = Math.max(0, game.shake - dt * 30);
  updateHud();
}

function drawBackground() {
  ctx.fillStyle = '#071018';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  const grad = ctx.createRadialGradient(
    canvas.width / 2,
    canvas.height / 2,
    80,
    canvas.width / 2,
    canvas.height / 2,
    420,
  );
  grad.addColorStop(0, 'rgba(54, 95, 138, 0.35)');
  grad.addColorStop(1, 'rgba(15, 23, 31, 0.05)');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // grid lines
  ctx.strokeStyle = 'rgba(130, 168, 207, 0.08)';
  ctx.lineWidth = 1;
  for (let x = 0; x <= canvas.width; x += 40) {
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x, canvas.height);
    ctx.stroke();
  }
  for (let y = 0; y <= canvas.height; y += 40) {
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(canvas.width, y);
    ctx.stroke();
  }
}

function drawWalls() {
  for (const wall of walls) {
    ctx.fillStyle = '#1c2d3c';
    ctx.fillRect(wall.x, wall.y, wall.w, wall.h);
    ctx.strokeStyle = '#8ad4ff';
    ctx.strokeRect(wall.x + 0.5, wall.y + 0.5, wall.w - 1, wall.h - 1);
  }
}

function drawPlayer() {
  ctx.save();
  ctx.translate(player.x, player.y);
  ctx.rotate(player.angle);

  ctx.fillStyle = player.flash > 0 ? '#bff6ff' : '#7ef6ff';
  ctx.fillRect(0, -5, 22, 10);

  ctx.fillStyle = '#0d1d2b';
  ctx.fillRect(-6, -6, 12, 12);

  ctx.restore();

  ctx.beginPath();
  ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
  ctx.strokeStyle = 'rgba(126, 246, 255, 0.6)';
  ctx.lineWidth = 2;
  ctx.stroke();
}

function drawBullets() {
  for (const bullet of bullets) {
    ctx.beginPath();
    ctx.arc(bullet.x, bullet.y, bullet.r, 0, Math.PI * 2);
    ctx.fillStyle = '#d8fbff';
    ctx.fill();
  }
}

function drawEnemies() {
  for (const enemy of enemies) {
    ctx.beginPath();
    ctx.arc(enemy.x, enemy.y, enemy.r, 0, Math.PI * 2);
    ctx.fillStyle = enemy.color;
    ctx.fill();

    // health bar
    const barWidth = enemy.r * 2;
    ctx.fillStyle = 'rgba(0, 0, 0, 0.4)';
    ctx.fillRect(enemy.x - barWidth / 2, enemy.y - enemy.r - 12, barWidth, 4);
    ctx.fillStyle = '#7ef6ff';
    ctx.fillRect(
      enemy.x - barWidth / 2,
      enemy.y - enemy.r - 12,
      (enemy.hp / enemy.maxHp) * barWidth,
      4,
    );
  }
}

function drawParticles() {
  for (const p of particles) {
    ctx.fillStyle = p.color;
    ctx.globalAlpha = Math.max(0, p.life);
    ctx.fillRect(p.x, p.y, p.size, p.size);
    ctx.globalAlpha = 1;
  }
}

function drawCrosshair() {
  const x = input.mouseX;
  const y = input.mouseY;

  ctx.strokeStyle = 'rgba(255,255,255,0.9)';
  ctx.lineWidth = 1.4;
  ctx.beginPath();
  ctx.moveTo(x - 8, y);
  ctx.lineTo(x + 8, y);
  ctx.moveTo(x, y - 8);
  ctx.lineTo(x, y + 8);
  ctx.stroke();

  ctx.strokeStyle = 'rgba(61, 229, 255, 0.9)';
  ctx.beginPath();
  ctx.arc(x, y, 12, 0, Math.PI * 2);
  ctx.stroke();
}

function drawHUD() {
  if (player.reloadTimer > 0) {
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(0, canvas.height - 18, canvas.width, 18);
    ctx.fillStyle = '#ffe66d';
    ctx.fillRect(0, canvas.height - 18, (1 - player.reloadTimer / player.reloadTime) * canvas.width, 18);
  }

  // health bar
  ctx.fillStyle = 'rgba(10, 15, 20, 0.7)';
  ctx.fillRect(16, 16, 180, 20);
  ctx.fillStyle = '#ff6d6d';
  ctx.fillRect(16, 16, (player.hp / player.maxHp) * 180, 20);
  ctx.strokeStyle = 'rgba(255,255,255,0.5)';
  ctx.strokeRect(16, 16, 180, 20);

  // minimap style
  const mapX = canvas.width - 110;
  const mapY = canvas.height - 110;
  ctx.fillStyle = 'rgba(8,14,18,0.7)';
  ctx.fillRect(mapX, mapY, 90, 90);
  ctx.strokeStyle = 'rgba(100,204,255,0.4)';
  ctx.strokeRect(mapX, mapY, 90, 90);
  ctx.fillStyle = '#7ef6ff';
  ctx.fillRect(player.x / world.width * 90 + mapX - 2, player.y / world.height * 90 + mapY - 2, 4, 4);
}

function draw() {
  ctx.save();
  if (game.shake > 0) {
    ctx.translate((Math.random() - 0.5) * game.shake, (Math.random() - 0.5) * game.shake);
  }
  drawBackground();
  drawWalls();
  drawBullets();
  drawEnemies();
  drawPlayer();
  drawParticles();
  drawCrosshair();
  drawHUD();
  ctx.restore();
}

function loop(timestamp) {
  const dt = Math.min((timestamp - game.lastTime) / 1000 || 0.016, 0.032);
  game.lastTime = timestamp;

  update(dt);
  draw();

  requestAnimationFrame(loop);
}

window.addEventListener('keydown', (event) => {
  input.keys[event.code] = true;
  if (event.code === 'KeyR') reload();
});

window.addEventListener('keyup', (event) => {
  input.keys[event.code] = false;
});

canvas.addEventListener('mousemove', (event) => {
  const rect = canvas.getBoundingClientRect();
  const scaleX = canvas.width / rect.width;
  const scaleY = canvas.height / rect.height;
  input.mouseX = (event.clientX - rect.left) * scaleX;
  input.mouseY = (event.clientY - rect.top) * scaleY;
});

canvas.addEventListener('mousedown', () => {
  if (!game.running) return;
  input.isMouseDown = true;
  shoot();
});

canvas.addEventListener('mouseup', () => {
  input.isMouseDown = false;
});

startButton.addEventListener('click', () => {
  overlay.classList.remove('visible');
  startGame();
  updateHud();
});

resetGame();
updateHud();
requestAnimationFrame(loop);

window.addEventListener('blur', () => {
  input.keys = {};
  input.isMouseDown = false;
});

// seed one enemy on menu so the area feels active
for (let i = 0; i < 3; i += 1) {
  spawnEnemy();
}
