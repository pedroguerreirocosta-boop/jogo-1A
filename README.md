# jogo-1A
let player;
let obstacles = [];
let portals = [];
let particles = [];
let speed = 8;
let baseSpeed = 8;
let score = 0;
let gameState = 'MENU';
let currentLevel = 1;

// Spawn timing
let framesUntilNextObstacle = 0;
let framesUntilNextPortal = 0;
let spawnRateMin = 45;
let spawnRateMax = 80;
let portalRateMin = 240;
let portalRateMax = 420;

// Estado global do jogador
let gravityDir = 1;
let mode = 'CUBE';

const FLOOR_Y = 370;
const CEILING_Y = 30;

// ==========================================
// ENHANCED GRAPHICS GLOBALS
// ==========================================
let bgStars = [];
let bgPulse = 0;
let screenShakeAmount = 0;
let screenShakeDecay = 0.9;
let trailHistory = [];
const MAX_TRAIL = 18;
let groundParticles = [];
let speedLines = [];
let menuPulse = 0;
let titleGlow = 0;
let levelColors = {
  1: { primary: [0, 255, 150], secondary: [0, 180, 255], bg: [15, 15, 30] },
  2: { primary: [255, 200, 50], secondary: [255, 100, 0], bg: [20, 12, 25] },
  3: { primary: [255, 60, 90], secondary: [255, 0, 200], bg: [25, 10, 15] },
  4: { primary: [200, 50, 255], secondary: [255, 0, 100], bg: [18, 5, 25] }
};
let flashAlpha = 0;
let flashColor = [255, 255, 255];
let comboCounter = 0;
let comboTimer = 0;
let scorePopups = [];
let waveOffset = 0;

// Glow buffer
let glowLayer;

function setup() {
  createCanvas(800, 400);
  glowLayer = createGraphics(800, 400);
  player = new Player();
  initStars();
}

function initStars() {
  bgStars = [];
  for (let i = 0; i < 80; i++) {
    bgStars.push({
      x: random(width),
      y: random(height),
      size: random(1, 3.5),
      speed: random(0.2, 1.5),
      brightness: random(80, 255),
      twinkleSpeed: random(0.02, 0.08),
      twinkleOffset: random(TWO_PI)
    });
  }
}

function draw() {
  let lc = levelColors[currentLevel] || levelColors[1];

  // Screen shake
  push();
  if (screenShakeAmount > 0.5) {
    translate(random(-screenShakeAmount, screenShakeAmount), random(-screenShakeAmount, screenShakeAmount));
    screenShakeAmount *= screenShakeDecay;
  } else {
    screenShakeAmount = 0;
  }

  // Dynamic background
  let bgR = lc.bg[0] + sin(bgPulse * 0.3) * 5;
  let bgG = lc.bg[1] + sin(bgPulse * 0.25) * 5;
  let bgB = lc.bg[2] + sin(bgPulse * 0.2) * 5;
  background(bgR, bgG, bgB);
  bgPulse += 0.02;

  drawEnhancedBackground(lc);

  if (gameState === 'MENU') {
    drawEnhancedMenu(lc);
  } else if (gameState === 'PLAYING') {
    playGame(lc);
  } else if (gameState === 'GAMEOVER') {
    drawEnhancedGameOver(lc);
  }

  // Flash overlay
  if (flashAlpha > 1) {
    noStroke();
    fill(flashColor[0], flashColor[1], flashColor[2], flashAlpha);
    rect(0, 0, width, height);
    flashAlpha *= 0.85;
  }

  pop();
}

// ==========================================
// ENHANCED BACKGROUND
// ==========================================
function drawEnhancedBackground(lc) {
  // Parallax stars
  for (let s of bgStars) {
    if (gameState === 'PLAYING') {
      s.x -= s.speed * (speed / 8);
      if (s.x < 0) { s.x = width; s.y = random(height); }
    }
    let twinkle = sin(frameCount * s.twinkleSpeed + s.twinkleOffset);
    let alpha = s.brightness * map(twinkle, -1, 1, 0.3, 1);
    let sz = s.size * map(twinkle, -1, 1, 0.7, 1.2);

    noStroke();
    // Star glow
    fill(lc.primary[0], lc.primary[1], lc.primary[2], alpha * 0.15);
    ellipse(s.x, s.y, sz * 4, sz * 4);
    fill(255, 255, 255, alpha);
    ellipse(s.x, s.y, sz, sz);
  }

  // Animated grid with depth
  stroke(lc.primary[0], lc.primary[1], lc.primary[2], 18);
  strokeWeight(0.5);
  let gridOffset = gameState === 'PLAYING' ? (frameCount * speed * 0.5) % 40 : (frameCount * 0.5) % 40;
  for (let x = -gridOffset; x < width + 40; x += 40) {
    line(x, 0, x, height);
  }
  for (let y = 0; y < height; y += 40) {
    line(0, y, width, y);
  }

  // Secondary deeper grid (parallax)
  stroke(lc.secondary[0], lc.secondary[1], lc.secondary[2], 8);
  let gridOffset2 = gameState === 'PLAYING' ? (frameCount * speed * 0.25) % 80 : (frameCount * 0.25) % 80;
  for (let x = -gridOffset2; x < width + 80; x += 80) {
    line(x, 0, x, height);
  }
  noStroke();
}

// ==========================================
// ENHANCED MENU
// ==========================================
function drawEnhancedMenu(lc) {
  menuPulse += 0.04;
  titleGlow += 0.06;

  // Title with glow
  let glowSize = 3 + sin(titleGlow) * 2;
  textAlign(CENTER);
  textSize(48);
  textStyle(BOLD);

  // Glow layers
  for (let i = 3; i > 0; i--) {
    fill(lc.primary[0], lc.primary[1], lc.primary[2], 30 / i);
    text("GEOMETRY DASH", width / 2, 88 + sin(menuPulse) * 3);
  }
  fill(lc.primary[0], lc.primary[1], lc.primary[2]);
  text("GEOMETRY DASH", width / 2, 88 + sin(menuPulse) * 3);

  // Subtitle
  textStyle(NORMAL);
  textSize(16);
  fill(200, 200, 255, 180);
  text("Com portais de gravidade, velocidade e modo nave!", width / 2, 118);

  // Difficulty buttons with hover-like glow
  textSize(20);
  let difficulties = [
    { key: '1', label: 'FACIL', color: [100, 255, 100], y: 185 },
    { key: '2', label: 'MEDIO', color: [255, 255, 100], y: 220 },
    { key: '3', label: 'DIFICIL', color: [255, 130, 130], y: 255 },
    { key: '4', label: 'EXTREMO', color: [255, 80, 200], y: 290 }
  ];

  for (let d of difficulties) {
    let pulse = sin(menuPulse * 2 + d.y * 0.05) * 0.3 + 0.7;

    // Button background
    noStroke();
    fill(d.color[0], d.color[1], d.color[2], 15 * pulse);
    rect(width / 2 - 120, d.y - 18, 240, 28, 14);

    // Glow border
    stroke(d.color[0], d.color[1], d.color[2], 60 * pulse);
    strokeWeight(1.5);
    noFill();
    rect(width / 2 - 120, d.y - 18, 240, 28, 14);

    // Text
    noStroke();
    fill(d.color[0], d.color[1], d.color[2], 200 + 55 * pulse);
    textAlign(CENTER);
    text("[" + d.key + "] " + d.label, width / 2, d.y + 2);
  }

  // Controls info
  fill(150, 160, 200, 150 + sin(menuPulse * 3) * 30);
  textSize(13);
  text("Cubo: ESPACO/\u2191 pra pular  \u2022  Nave: segure ESPACO/\u2191 pra subir", width / 2, 340);

  // Animated particles in menu
  if (frameCount % 4 === 0) {
    particles.push(new Particle(
      random(width), height + 5,
      color(lc.primary[0], lc.primary[1], lc.primary[2], 120),
      random(-0.5, 0.5), random(-2, -4)
    ));
  }

  for (let i = particles.length - 1; i >= 0; i--) {
    particles[i].update();
    particles[i].show();
    if (particles[i].dead()) particles.splice(i, 1);
  }
}

// ==========================================
// ENHANCED GAME OVER
// ==========================================
function drawEnhancedGameOver(lc) {
  // Dark overlay
  noStroke();
  fill(0, 0, 0, 150);
  rect(0, 0, width, height);

  // Vignette
  drawVignette(200);

  textAlign(CENTER);

  // GAME OVER with shake effect
  let shk = max(screenShakeAmount * 0.3, 0);
  textSize(52);
  textStyle(BOLD);
  // Red glow
  for (let i = 4; i > 0; i--) {
    fill(255, 40, 40, 40 / i);
    text("GAME OVER", width / 2 + random(-shk, shk), height / 2 - 40 + random(-shk, shk));
  }
  fill(255, 60, 60);
  text("GAME OVER", width / 2, height / 2 - 40);

  textStyle(NORMAL);
  textSize(22);
  fill(255, 220);
  text("Pontuacao Final: " + score, width / 2, height / 2 + 10);

  textSize(16);
  fill(180, 200, 255, 180 + sin(frameCount * 0.06) * 50);
  text("Pressione 'R' para tentar novamente", width / 2, height / 2 + 50);
  text("Pressione 'M' para voltar ao Menu", width / 2, height / 2 + 75);

  // Still render particles
  for (let i = particles.length - 1; i >= 0; i--) {
    particles[i].update();
    particles[i].show();
    if (particles[i].dead()) particles.splice(i, 1);
  }
}

// ==========================================
// PLAY GAME (ENHANCED)
// ==========================================
function playGame(lc) {
  waveOffset += 0.05;

  // Enhanced ground and ceiling
  drawEnhancedBoundaries(lc);

  // Speed lines effect
  if (speed > 8) {
    drawSpeedLines(lc);
  }

  // Player
  player.update(lc);
  player.show(lc);

  // Obstacles
  framesUntilNextObstacle--;
  if (framesUntilNextObstacle <= 0) {
    spawnObstacle();
    framesUntilNextObstacle = floor(random(spawnRateMin, spawnRateMax));
  }

  // Portals
  framesUntilNextPortal--;
  if (framesUntilNextPortal <= 0) {
    portals.push(new Portal(randomPortalType()));
    framesUntilNextPortal = floor(random(portalRateMin, portalRateMax));
  }

  // Update + render obstacles
  for (let i = obstacles.length - 1; i >= 0; i--) {
    obstacles[i].update();
    obstacles[i].show(lc);

    if (player.hits(obstacles[i])) {
      gameState = 'GAMEOVER';
      spawnEnhancedExplosion(player.x + player.size / 2, player.y + player.size / 2);
      screenShakeAmount = 20;
      flashAlpha = 120;
      flashColor = [255, 50, 50];
    }

    if (obstacles[i].offscreen()) {
      obstacles.splice(i, 1);
      score++;
      comboCounter++;
      comboTimer = 60;
      // Score popup
      scorePopups.push({
        x: 120 + random(-10, 10),
        y: 55,
        text: '+1',
        life: 40,
        vy: -1.5,
        color: lc.primary
      });
      if (score % 5 === 0) speed += 0.3;
    }
  }

  // Update + render portals
  for (let i = portals.length - 1; i >= 0; i--) {
    portals[i].update();
    portals[i].show();

    if (player.hitsPortal(portals[i]) && !portals[i].used) {
      portals[i].apply();
      portals[i].used = true;
      flashAlpha = 60;
      let pc = portals[i].colorA();
      flashColor = [red(pc), green(pc), blue(pc)];
      screenShakeAmount = 6;
    }

    if (portals[i].offscreen()) portals.splice(i, 1);
  }

  // Particles
  for (let i = particles.length - 1; i >= 0; i--) {
    particles[i].update();
    particles[i].show();
    if (particles[i].dead()) particles.splice(i, 1);
  }

  // Score popups
  for (let i = scorePopups.length - 1; i >= 0; i--) {
    let sp = scorePopups[i];
    sp.y += sp.vy;
    sp.life--;
    let alpha = map(sp.life, 0, 40, 0, 255);
    fill(sp.color[0], sp.color[1], sp.color[2], alpha);
    noStroke();
    textSize(14);
    textAlign(LEFT);
    text(sp.text, sp.x, sp.y);
    if (sp.life <= 0) scorePopups.splice(i, 1);
  }

  // Combo timer
  if (comboTimer > 0) comboTimer--;
  else comboCounter = 0;

  // Vignette
  drawVignette(80);

  // HUD
  drawEnhancedHUD(lc);
}

// ==========================================
// ENHANCED BOUNDARIES (GROUND/CEILING)
// ==========================================
function drawEnhancedBoundaries(lc) {
  // Ground glow
  for (let i = 5; i > 0; i--) {
    stroke(lc.primary[0], lc.primary[1], lc.primary[2], 15 * (6 - i));
    strokeWeight(i * 1.5);
    line(0, FLOOR_Y, width, FLOOR_Y);
  }

  // Ground wave particles
  for (let x = 0; x < width; x += 20) {
    let waveY = sin(x * 0.04 + waveOffset) * 3;
    let alpha = 30 + sin(x * 0.02 + waveOffset * 2) * 15;
    noStroke();
    fill(lc.primary[0], lc.primary[1], lc.primary[2], alpha);
    ellipse(x, FLOOR_Y + waveY, 3, 3);
  }

  // Ceiling glow
  for (let i = 5; i > 0; i--) {
    stroke(lc.secondary[0], lc.secondary[1], lc.secondary[2], 12 * (6 - i));
    strokeWeight(i * 1.5);
    line(0, CEILING_Y, width, CEILING_Y);
  }

  // Ceiling wave
  for (let x = 0; x < width; x += 20) {
    let waveY = sin(x * 0.04 + waveOffset + PI) * 3;
    let alpha = 25 + sin(x * 0.02 + waveOffset * 2) * 12;
    noStroke();
    fill(lc.secondary[0], lc.secondary[1], lc.secondary[2], alpha);
    ellipse(x, CEILING_Y + waveY, 3, 3);
  }

  noStroke();
}

// ==========================================
// SPEED LINES
// ==========================================
function drawSpeedLines(lc) {
  let intensity = map(speed, 8, 22, 0, 1);
  if (frameCount % max(1, floor(4 - intensity * 3)) === 0) {
    speedLines.push({
      x: width + 10,
      y: random(CEILING_Y + 10, FLOOR_Y - 10),
      len: random(30, 80) * intensity,
      alpha: random(20, 60) * intensity,
      speed: speed * random(1.5, 2.5)
    });
  }

  for (let i = speedLines.length - 1; i >= 0; i--) {
    let sl = speedLines[i];
    sl.x -= sl.speed;
    stroke(lc.primary[0], lc.primary[1], lc.primary[2], sl.alpha);
    strokeWeight(1);
    line(sl.x, sl.y, sl.x + sl.len, sl.y);
    if (sl.x + sl.len < 0) speedLines.splice(i, 1);
  }
  noStroke();
}

// ==========================================
// VIGNETTE
// ==========================================
function drawVignette(intensity) {
  noStroke();
  // Top
  for (let i = 0; i < 60; i++) {
    fill(0, 0, 0, map(i, 0, 60, intensity, 0));
    rect(0, i, width, 1);
  }
  // Bottom
  for (let i = 0; i < 60; i++) {
    fill(0, 0, 0, map(i, 0, 60, 0, intensity));
    rect(0, height - 60 + i, width, 1);
  }
  // Sides
  for (let i = 0; i < 40; i++) {
    fill(0, 0, 0, map(i, 0, 40, intensity * 0.5, 0));
    rect(i, 0, 1, height);
    rect(width - i, 0, 1, height);
  }
}

// ==========================================
// ENHANCED HUD
// ==========================================
function drawEnhancedHUD(lc) {
  noStroke();

  // Score with glow
  textSize(24);
  textAlign(LEFT);
  textStyle(BOLD);
  fill(lc.primary[0], lc.primary[1], lc.primary[2], 40);
  text("Pontos: " + score, 21, 61);
  fill(255);
  text("Pontos: " + score, 20, 60);

  textSize(16);
  textStyle(NORMAL);
  fill(200, 220, 255, 180);
  text("Fase: " + currentLevel, 20, 82);

  // Combo display
  if (comboCounter > 2) {
    let comboAlpha = map(comboTimer, 0, 60, 100, 255);
    let comboScale = 1 + sin(frameCount * 0.15) * 0.1;
    textSize(14 * comboScale);
    fill(255, 220, 50, comboAlpha);
    text("Combo x" + comboCounter, 20, 100);
  }

  // Right side HUD
  textAlign(RIGHT);
  textSize(15);

  // Mode indicator with icon color
  if (mode === 'SHIP') {
    fill(0, 220, 255, 220);
  } else {
    fill(lc.primary[0], lc.primary[1], lc.primary[2], 220);
  }
  text("Modo: " + mode, width - 20, 58);

  // Gravity indicator
  fill(gravityDir === 1 ? color(180, 220, 255, 200) : color(255, 230, 60, 200));
  text("Gravidade: " + (gravityDir === 1 ? "\u2193" : "\u2191"), width - 20, 76);

  // Speed bar
  let speedPercent = map(speed, 4, 22, 0, 1);
  let barW = 100;
  let barH = 8;
  let barX = width - 20 - barW;
  let barY = 86;

  // Bar background
  fill(40, 40, 60, 150);
  rect(barX, barY, barW, barH, 4);

  // Bar fill with gradient color
  let speedColor = lerpColor(color(0, 255, 150), color(255, 50, 50), speedPercent);
  fill(speedColor);
  rect(barX, barY, barW * speedPercent, barH, 4);

  // Speed text
  fill(200, 220, 255, 180);
  textSize(12);
  text("Vel: " + speed.toFixed(1), width - 20, barY + barH + 14);
}

// ==========================================
// SPAWNING
// ==========================================
function spawnObstacle() {
  let types = ['spike'];
  if (currentLevel >= 2) types.push('tall', 'double');
  if (currentLevel >= 3) types.push('triple', 'block');
  if (currentLevel >= 4) types.push('ceiling', 'gap');

  let type = random(types);
  if (type === 'spike') {
    obstacles.push(new Obstacle(width, 'spike', 40, 40));
  } else if (type === 'tall') {
    obstacles.push(new Obstacle(width, 'spike', 40, 60));
  } else if (type === 'double') {
    obstacles.push(new Obstacle(width, 'spike', 40, 40));
    obstacles.push(new Obstacle(width + 50, 'spike', 40, 40));
  } else if (type === 'triple') {
    for (let k = 0; k < 3; k++) {
      obstacles.push(new Obstacle(width + k * 45, 'spike', 40, 40));
    }
  } else if (type === 'block') {
    obstacles.push(new Obstacle(width, 'block', 50, 60));
  } else if (type === 'ceiling') {
    obstacles.push(new Obstacle(width, 'ceiling', 40, 50));
  } else if (type === 'gap') {
    obstacles.push(new Obstacle(width, 'spike', 40, 40));
    obstacles.push(new Obstacle(width, 'ceiling', 40, 40));
  }
}

function randomPortalType() {
  let pool = ['gravity', 'speed_up', 'speed_down'];
  if (currentLevel >= 2) pool.push('ship', 'cube');
  return random(pool);
}

// ==========================================
// CONTROLS
// ==========================================
function keyPressed() {
  if (gameState === 'MENU') {
    if (key === '1') startGame(1);
    if (key === '2') startGame(2);
    if (key === '3') startGame(3);
    if (key === '4') startGame(4);
  } else if (gameState === 'PLAYING') {
    if ((key === ' ' || keyCode === UP_ARROW) && mode === 'CUBE') {
      player.jump();
    }
  } else if (gameState === 'GAMEOVER') {
    if (key === 'r' || key === 'R') startGame(currentLevel);
    if (key === 'm' || key === 'M') {
      gameState = 'MENU';
      particles = [];
      speedLines = [];
      scorePopups = [];
    }
  }
}

function startGame(level) {
  player = new Player();
  obstacles = [];
  portals = [];
  particles = [];
  speedLines = [];
  scorePopups = [];
  score = 0;
  currentLevel = level;
  gameState = 'PLAYING';
  gravityDir = 1;
  mode = 'CUBE';
  screenShakeAmount = 0;
  comboCounter = 0;
  comboTimer = 0;
  trailHistory = [];

  if (level === 1) {
    speed = 5.5;
    spawnRateMin = 75; spawnRateMax = 115;
    portalRateMin = 360; portalRateMax = 600;
  } else if (level === 2) {
    speed = 7.5;
    spawnRateMin = 50; spawnRateMax = 85;
    portalRateMin = 280; portalRateMax = 460;
  } else if (level === 3) {
    speed = 10;
    spawnRateMin = 32; spawnRateMax = 58;
    portalRateMin = 220; portalRateMax = 360;
  } else if (level === 4) {
    speed = 13;
    spawnRateMin = 22; spawnRateMax = 40;
    portalRateMin = 160; portalRateMax = 280;
  }
  baseSpeed = speed;

  framesUntilNextObstacle = floor(random(spawnRateMin, spawnRateMax));
  framesUntilNextPortal = floor(random(portalRateMin * 0.5, portalRateMax * 0.5));
}

// ==========================================
// PLAYER (ENHANCED)
// ==========================================
class Player {
  constructor() {
    this.size = 36;
    this.x = 100;
    this.y = FLOOR_Y - this.size;
    this.velocity = 0;
    this.gravity = 1.1;
    this.jumpForce = 16;
    this.rotation = 0;
    this.jumpSquash = 1;
    this.landSquash = 1;
    this.wasOnGround = true;
  }

  jump() {
    if (this.onGround()) {
      this.velocity = -this.jumpForce * gravityDir;
      this.jumpSquash = 0.6;
      // Jump particles burst
      let lc = levelColors[currentLevel] || levelColors[1];
      for (let i = 0; i < 8; i++) {
        particles.push(new Particle(
          this.x + this.size / 2 + random(-8, 8),
          gravityDir === 1 ? this.y + this.size : this.y,
          color(lc.primary[0], lc.primary[1], lc.primary[2], 180),
          random(-2, 2),
          random(1, 3) * gravityDir
        ));
      }
    }
  }

  onGround() {
    if (gravityDir === 1) return this.y >= FLOOR_Y - this.size - 0.5;
    return this.y <= CEILING_Y + 0.5;
  }

  update(lc) {
    if (mode === 'CUBE') {
      this.velocity += this.gravity * gravityDir;
      this.y += this.velocity;

      if (gravityDir === 1 && this.y >= FLOOR_Y - this.size) {
        this.y = FLOOR_Y - this.size;
        if (abs(this.velocity) > 3) {
          this.landSquash = 1.3;
          // Landing dust
          for (let i = 0; i < 5; i++) {
            particles.push(new Particle(
              this.x + this.size / 2 + random(-12, 12),
              FLOOR_Y,
              color(lc.primary[0], lc.primary[1], lc.primary[2], 100),
              random(-1.5, 1.5), random(-1.5, -0.3)
            ));
          }
        }
        this.velocity = 0;
      }
      if (gravityDir === -1 && this.y <= CEILING_Y) {
        this.y = CEILING_Y;
        if (abs(this.velocity) > 3) {
          this.landSquash = 1.3;
          for (let i = 0; i < 5; i++) {
            particles.push(new Particle(
              this.x + this.size / 2 + random(-12, 12),
              CEILING_Y,
              color(lc.primary[0], lc.primary[1], lc.primary[2], 100),
              random(-1.5, 1.5), random(0.3, 1.5)
            ));
          }
        }
        this.velocity = 0;
      }

      if (!this.onGround()) {
        this.rotation += 0.18 * gravityDir;
      } else {
        let snap = round(this.rotation / (PI / 2)) * (PI / 2);
        this.rotation = lerp(this.rotation, snap, 0.5);
      }
    } else if (mode === 'SHIP') {
      let holding = keyIsDown(32) || keyIsDown(UP_ARROW);
      let thrust = holding ? -0.7 : 0.7;
      this.velocity += thrust * gravityDir;
      this.velocity = constrain(this.velocity, -9, 9);
      this.y += this.velocity;

      if (this.y >= FLOOR_Y - this.size) {
        this.y = FLOOR_Y - this.size;
        this.velocity = 0;
      }
      if (this.y <= CEILING_Y) {
        this.y = CEILING_Y;
        this.velocity = 0;
      }

      this.rotation = map(this.velocity, -9, 9, -PI / 6, PI / 6);

      // Ship engine particles
      if (holding && frameCount % 2 === 0) {
        for (let i = 0; i < 2; i++) {
          particles.push(new Particle(
            this.x - 5,
            this.y + this.size / 2 + random(-5, 5),
            color(0, 200, 255, 200),
            random(-4, -2), random(-1, 1)
          ));
        }
      }
    }

    // Squash & stretch lerp
    this.jumpSquash = lerp(this.jumpSquash, 1, 0.15);
    this.landSquash = lerp(this.landSquash, 1, 0.12);

    // Trail history for afterimage
    if (frameCount % 2 === 0) {
      trailHistory.push({
        x: this.x, y: this.y,
        rotation: this.rotation,
        mode: mode, life: MAX_TRAIL
      });
      if (trailHistory.length > MAX_TRAIL) trailHistory.shift();
    }

    // Trail particles
    if (frameCount % 2 === 0) {
      particles.push(new Particle(
        this.x + (mode === 'SHIP' ? 0 : this.size / 2),
        this.y + this.size / 2,
        color(lc.primary[0], lc.primary[1], lc.primary[2], 150),
        random(-1, -3), random(-0.5, 0.5)
      ));
    }
  }

  show(lc) {
    // Draw afterimage trail
    for (let i = 0; i < trailHistory.length; i++) {
      let t = trailHistory[i];
      let alpha = map(i, 0, trailHistory.length, 5, 40);
      let sc = map(i, 0, trailHistory.length, 0.6, 0.95);

      push();
      translate(t.x + this.size / 2, t.y + this.size / 2);
      rotate(t.rotation);
      scale(sc);
      noStroke();

      if (t.mode === 'CUBE') {
        fill(lc.primary[0], lc.primary[1], lc.primary[2], alpha);
        rect(-this.size / 2, -this.size / 2, this.size, this.size, 5);
      } else {
        fill(0, 220, 255, alpha);
        let s = this.size;
        beginShape();
        vertex(-s / 2, -s / 3);
        vertex(s / 2, 0);
        vertex(-s / 2, s / 3);
        endShape(CLOSE);
      }
      pop();
    }

    push();
    translate(this.x + this.size / 2, this.y + this.size / 2);
    rotate(this.rotation);

    // Squash & stretch
    let sx = this.jumpSquash * (1 / this.landSquash);
    let sy = (1 / this.jumpSquash) * this.landSquash;
    scale(sx, sy);

    if (mode === 'CUBE') {
      // Outer glow
      noStroke();
      for (let i = 4; i > 0; i--) {
        fill(lc.primary[0], lc.primary[1], lc.primary[2], 12 * (5 - i));
        rect(-this.size / 2 - i * 2, -this.size / 2 - i * 2,
          this.size + i * 4, this.size + i * 4, 5 + i);
      }

      // Main cube
      fill(lc.primary[0], lc.primary[1], lc.primary[2]);
      stroke(255, 230);
      strokeWeight(2);
      rect(-this.size / 2, -this.size / 2, this.size, this.size, 5);

      // Inner detail with gradient feel
      noStroke();
      fill(255, 240);
      rect(-this.size / 4, -this.size / 4, this.size / 2, this.size / 2, 3);

      // Highlight
      fill(255, 255, 255, 80);
      rect(-this.size / 2 + 3, -this.size / 2 + 3, this.size - 6, this.size / 3, 3);
    } else {
      // SHIP mode
      let s = this.size;

      // Outer glow
      noStroke();
      for (let i = 4; i > 0; i--) {
        fill(0, 220, 255, 10 * (5 - i));
        beginShape();
        vertex(-s / 2 - i * 2, -s / 3 - i * 2);
        vertex(s / 2 + i * 2, 0);
        vertex(-s / 2 - i * 2, s / 3 + i * 2);
        endShape(CLOSE);
      }

      // Main ship
      fill(0, 220, 255);
      stroke(255, 230);
      strokeWeight(2);
      beginShape();
      vertex(-s / 2, -s / 3);
      vertex(s / 2, 0);
      vertex(-s / 2, s / 3);
      endShape(CLOSE);

      // Cockpit
      noStroke();
      fill(255, 240);
      ellipse(0, 0, s / 3, s / 4);

      // Highlight
      fill(255, 255, 255, 60);
      ellipse(-2, -3, s / 4, s / 6);
    }
    pop();
  }

  hits(obs) {
    return obs.collides(this);
  }

  hitsPortal(p) {
    return (
      this.x + this.size > p.x &&
      this.x < p.x + p.w &&
      this.y + this.size > p.y &&
      this.y < p.y + p.h
    );
  }
}

// ==========================================
// OBSTACLES (ENHANCED)
// ==========================================
class Obstacle {
  constructor(x, type, w, h) {
    this.type = type;
    this.w = w;
    this.h = h;
    this.x = x;
    if (type === 'ceiling') {
      this.y = CEILING_Y;
    } else {
      this.y = FLOOR_Y - h;
    }
    this.pulse = random(TWO_PI);
  }

  update() {
    this.x -= speed;
    this.pulse += 0.1;
  }

  show(lc) {
    let glowIntensity = 0.5 + sin(this.pulse) * 0.3;

    if (this.type === 'spike' || this.type === 'tall') {
      // Glow
      noStroke();
      for (let i = 4; i > 0; i--) {
        fill(255, 60, 90, 10 * (5 - i) * glowIntensity);
        triangle(
          this.x - i * 2, this.y + this.h + i,
          this.x + this.w / 2, this.y - i * 2,
          this.x + this.w + i * 2, this.y + this.h + i
        );
      }

      // Main spike
      fill(255, 60, 90);
      stroke(255, 150, 170, 200);
      strokeWeight(1.5);
      triangle(
        this.x, this.y + this.h,
        this.x + this.w / 2, this.y,
        this.x + this.w, this.y + this.h
      );

      // Inner highlight
      noStroke();
      fill(255, 120, 150, 100);
      triangle(
        this.x + this.w * 0.2, this.y + this.h,
        this.x + this.w / 2, this.y + this.h * 0.35,
        this.x + this.w * 0.5, this.y + this.h
      );
    } else if (this.type === 'ceiling') {
      // Ceiling spike glow
      noStroke();
      for (let i = 4; i > 0; i--) {
        fill(255, 60, 90, 10 * (5 - i) * glowIntensity);
        triangle(
          this.x - i * 2, this.y - i,
          this.x + this.w / 2, this.y + this.h + i * 2,
          this.x + this.w + i * 2, this.y - i
        );
      }

      fill(255, 60, 90);
      stroke(255, 150, 170, 200);
      strokeWeight(1.5);
      triangle(
        this.x, this.y,
        this.x + this.w / 2, this.y + this.h,
        this.x + this.w, this.y
      );

      noStroke();
      fill(255, 120, 150, 100);
      triangle(
        this.x + this.w * 0.2, this.y,
        this.x + this.w / 2, this.y + this.h * 0.65,
        this.x + this.w * 0.5, this.y
      );
    } else if (this.type === 'block') {
      // Block glow
      noStroke();
      for (let i = 4; i > 0; i--) {
        fill(255, 200, 60, 10 * (5 - i) * glowIntensity);
        rect(this.x - i * 2, this.y - i * 2,
          this.w + i * 4, this.h + i * 4, 4 + i);
      }

      fill(255, 200, 60);
      stroke(255, 230, 150, 200);
      strokeWeight(2);
      rect(this.x, this.y, this.w, this.h, 4);

      // Inner X pattern
      noStroke();
      fill(255, 240, 180, 80);
      rect(this.x + 5, this.y + 5, this.w - 10, this.h - 10, 3);

      stroke(255, 200, 60, 150);
      strokeWeight(1.5);
      line(this.x + 8, this.y + 8, this.x + this.w - 8, this.y + this.h - 8);
      line(this.x + this.w - 8, this.y + 8, this.x + 8, this.y + this.h - 8);
    }
    noStroke();

    // Warning glow on ground below obstacles
    if (this.type !== 'ceiling' && this.x > 0 && this.x < width) {
      fill(255, 60, 90, 15 * glowIntensity);
      rect(this.x - 2, FLOOR_Y - 3, this.w + 4, 3, 2);
    }
  }

  collides(player) {
    let pL = player.x, pR = player.x + player.size;
    let pT = player.y, pB = player.y + player.size;

    if (this.type === 'block') {
      return (pR > this.x && pL < this.x + this.w &&
        pB > this.y && pT < this.y + this.h);
    }
    let pad = this.w * 0.15;
    let oL = this.x + pad, oR = this.x + this.w - pad;
    let oT = this.y + pad, oB = this.y + this.h - pad;
    return (pR > oL && pL < oR && pB > oT && pT < oB);
  }

  offscreen() {
    return this.x < -this.w - 10;
  }
}

// ==========================================
// PORTALS (ENHANCED)
// ==========================================
class Portal {
  constructor(type) {
    this.type = type;
    this.w = 30;
    this.h = 80;
    this.x = width;
    this.y = (FLOOR_Y + CEILING_Y) / 2 - this.h / 2;
    this.used = false;
    this.pulse = 0;
    this.portalParticles = [];
  }

  apply() {
    if (this.type === 'gravity') {
      gravityDir *= -1;
      player.velocity = 0;
    } else if (this.type === 'speed_up') {
      speed = min(speed + 3, 22);
    } else if (this.type === 'speed_down') {
      speed = max(speed - 3, 4);
    } else if (this.type === 'ship') {
      mode = 'SHIP';
    } else if (this.type === 'cube') {
      mode = 'CUBE';
    }
    spawnPortalBurst(this.x + this.w / 2, this.y + this.h / 2, this.colorA());
  }

  colorA() {
    if (this.type === 'gravity') return color(255, 230, 60);
    if (this.type === 'speed_up') return color(60, 255, 120);
    if (this.type === 'speed_down') return color(255, 140, 60);
    if (this.type === 'ship') return color(255, 90, 200);
    return color(255, 255, 255);
  }
  colorB() {
    if (this.type === 'gravity') return color(60, 130, 255);
    if (this.type === 'speed_up') return color(0, 200, 80);
    if (this.type === 'speed_down') return color(200, 80, 0);
    if (this.type === 'ship') return color(160, 60, 255);
    return color(120, 200, 255);
  }
  label() {
    if (this.type === 'gravity') return "\u2195";
    if (this.type === 'speed_up') return "\u00BB";
    if (this.type === 'speed_down') return "\u00AB";
    if (this.type === 'ship') return "\u25B2";
    if (this.type === 'cube') return "\u25A0";
    return "?";
  }

  update() {
    this.x -= speed;
    this.pulse += 0.15;

    // Portal ambient particles
    if (frameCount % 3 === 0 && this.x > -50 && this.x < width + 50) {
      let cA = this.colorA();
      particles.push(new Particle(
        this.x + random(0, this.w),
        this.y + random(0, this.h),
        color(red(cA), green(cA), blue(cA), 120),
        random(-1, 1), random(-2, 2)
      ));
    }
  }

  show() {
    if (this.used) return;

    push();
    let cx = this.x + this.w / 2;
    let cy = this.y + this.h / 2;

    // Outer glow aura
    let halo = 10 + sin(this.pulse) * 5;
    noStroke();
    let cA = this.colorA();
    for (let i = 4; i > 0; i--) {
      fill(red(cA), green(cA), blue(cA), 8 * (5 - i));
      rect(this.x - halo * i / 2, this.y - halo * i / 2,
        this.w + halo * i, this.h + halo * i, 12 + i * 2);
    }

    // Animated gradient fill
    for (let i = 0; i < this.h; i++) {
      let t = (i / this.h + sin(this.pulse + i * 0.05) * 0.1) % 1;
      let c = lerpColor(this.colorA(), this.colorB(), t);
      stroke(red(c), green(c), blue(c), 200);
      strokeWeight(1);
      line(this.x + 2, this.y + i, this.x + this.w - 2, this.y + i);
    }

    // Scanning line effect
    let scanY = this.y + ((frameCount * 3 + this.pulse * 10) % this.h);
    stroke(255, 255, 255, 150);
    strokeWeight(2);
    line(this.x, scanY, this.x + this.w, scanY);

    // Portal frame
    noFill();
    stroke(255, 230);
    strokeWeight(2.5);
    rect(this.x, this.y, this.w, this.h, 8);

    // Corner accents
    let cornerSize = 6;
    strokeWeight(3);
    stroke(255);
    // Top-left
    line(this.x, this.y + cornerSize, this.x, this.y);
    line(this.x, this.y, this.x + cornerSize, this.y);
    // Top-right
    line(this.x + this.w - cornerSize, this.y, this.x + this.w, this.y);
    line(this.x + this.w, this.y, this.x + this.w, this.y + cornerSize);
    // Bottom-left
    line(this.x, this.y + this.h - cornerSize, this.x, this.y + this.h);
    line(this.x, this.y + this.h, this.x + cornerSize, this.y + this.h);
    // Bottom-right
    line(this.x + this.w - cornerSize, this.y + this.h, this.x + this.w, this.y + this.h);
    line(this.x + this.w, this.y + this.h - cornerSize, this.x + this.w, this.y + this.h);

    // Icon with glow
    noStroke();
    fill(255, 255, 255, 60);
    textAlign(CENTER, CENTER);
    textSize(26);
    text(this.label(), cx, cy);
    fill(255);
    textSize(22);
    text(this.label(), cx, cy);

    pop();
  }

  offscreen() {
    return this.x < -this.w - 10;
  }
}

// ==========================================
// PARTICLES (ENHANCED)
// ==========================================
class Particle {
  constructor(x, y, c, vx, vy) {
    this.x = x;
    this.y = y;
    this.vx = vx !== undefined ? vx : random(-1, -3);
    this.vy = vy !== undefined ? vy : random(-1, 1);
    this.life = 30;
    this.maxLife = 30;
    this.c = c;
    this.size = random(2, 6);
    this.rotSpeed = random(-0.2, 0.2);
    this.rot = random(TWO_PI);
  }

  update() {
    this.x += this.vx;
    this.y += this.vy;
    this.vx *= 0.98;
    this.vy *= 0.98;
    this.life--;
    this.rot += this.rotSpeed;
    this.size *= 0.97;
  }

  show() {
    noStroke();
    let a = map(this.life, 0, this.maxLife, 0, 255);
    let r = red(this.c), g = green(this.c), b = blue(this.c);

    // Particle glow
    fill(r, g, b, a * 0.2);
    ellipse(this.x, this.y, this.size * 3, this.size * 3);

    // Main particle
    push();
    translate(this.x, this.y);
    rotate(this.rot);
    fill(r, g, b, a);
    rect(-this.size / 2, -this.size / 2, this.size, this.size, 1);
    pop();
  }

  dead() {
    return this.life <= 0 || this.size < 0.5;
  }
}

function spawnEnhancedExplosion(x, y) {
  // Main explosion
  for (let i = 0; i < 45; i++) {
    let angle = random(TWO_PI);
    let sp = random(2, 10);
    let colors = [
      color(255, 100, 100),
      color(255, 200, 50),
      color(255, 150, 0),
      color(255, 60, 60),
      color(255, 255, 200)
    ];
    let c = random(colors);
    let p = new Particle(x, y, c, cos(angle) * sp, sin(angle) * sp);
    p.maxLife = floor(random(30, 60));
    p.life = p.maxLife;
    p.size = random(3, 10);
    particles.push(p);
  }

  // Shockwave ring particles
  for (let i = 0; i < 20; i++) {
    let angle = (TWO_PI / 20) * i;
    let sp = 5;
    let p = new Particle(x, y, color(255, 255, 255, 200), cos(angle) * sp, sin(angle) * sp);
    p.size = 2;
    p.maxLife = 15;
    p.life = 15;
    particles.push(p);
  }
}

function spawnPortalBurst(x, y, c) {
  for (let i = 0; i < 25; i++) {
    let angle = random(TWO_PI);
    let sp = random(1, 6);
    let p = new Particle(x, y, c, cos(angle) * sp, sin(angle) * sp);
    p.maxLife = floor(random(20, 45));
    p.life = p.maxLife;
    p.size = random(3, 8);
    particles.push(p);
  }
}
