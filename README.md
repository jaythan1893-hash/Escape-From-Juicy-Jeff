
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Night Shift: Line of Sight Pathfinding</title>
    <style>
        * { box-sizing: border-box; touch-action: none; -webkit-user-select: none; user-select: none; }
        body {
            margin: 0;
            background: #050505;
            color: #fff;
            font-family: monospace;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            overflow: hidden;
        }
        #game-container {
            position: relative;
            width: 360px;
            height: 600px;
            background: #0f0f11;
            border: 3px solid #27272a;
            border-radius: 12px;
            overflow: hidden;
            box-shadow: 0 0 30px rgba(0,0,0,0.9);
        }
        canvas {
            width: 100%;
            height: 480px;
            background: #000;
            display: block;
        }
        .hud {
            position: absolute;
            top: 10px;
            left: 10px;
            display: flex;
            flex-direction: column;
            gap: 4px;
            font-size: 12px;
            font-weight: bold;
            pointer-events: auto;
            z-index: 15;
            background: rgba(0,0,0,0.85);
            border: 1px solid #3f3f46;
            padding: 8px;
            border-radius: 6px;
            letter-spacing: 1px;
        }
        .restart-top-btn {
            background: #ef4444;
            color: #fff;
            border: none;
            padding: 4px 8px;
            border-radius: 4px;
            font-family: inherit;
            font-weight: bold;
            font-size: 11px;
            cursor: pointer;
            text-align: center;
            margin-top: 2px;
        }

        .mobile-dpad {
            position: absolute;
            top: 10px;
            right: 10px;
            width: 120px;
            height: 120px;
            z-index: 15;
            background: rgba(24, 24, 27, 0.4);
            border: 2px solid rgba(63, 63, 70, 0.6);
            border-radius: 50%;
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            grid-template-rows: repeat(3, 1fr);
            backdrop-filter: blur(2px);
        }
        .dpad-btn {
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 16px;
            font-weight: bold;
            color: #a1a1aa;
            background: rgba(0, 0, 0, 0.3);
        }
        .dpad-btn:active, .dpad-btn.active {
            background: rgba(239, 68, 68, 0.6);
            color: #fff;
        }
        #dp-up { grid-column: 2; grid-row: 1; border-radius: 20px 20px 0 0; }
        #dp-left { grid-column: 1; grid-row: 2; border-radius: 20px 0 0 20px; }
        #dp-center { grid-column: 2; grid-row: 2; background: rgba(39, 39, 42, 0.8); border-radius: 50%; font-size: 10px; color: #52525b; }
        #dp-right { grid-column: 3; grid-row: 2; border-radius: 0 20px 20px 0; }
        #dp-down { grid-column: 2; grid-row: 3; border-radius: 0 0 20px 20px; }

        .controls {
            height: 120px;
            background: #09090b;
            display: flex;
            align-items: center;
            justify-content: space-around;
            padding: 10px;
            border-top: 2px solid #18181b;
        }
        .btn {
            width: 100px;
            padding: 12px;
            background: #18181b;
            color: #a1a1aa;
            border: 2px solid #27272a;
            border-radius: 8px;
            font-family: inherit;
            font-weight: bold;
            font-size: 12px;
            text-align: center;
            cursor: pointer;
            letter-spacing: 1px;
        }
        .btn.active { background: #7f1d1d; color: #fca5a5; border-color: #ef4444; }
        .btn.hide-active { background: #15803d; color: #86efac; border-color: #22c55e; }

        #overlay {
            display: none;
            position: absolute;
            inset: 0;
            background: rgba(0,0,0,0.95);
            flex-direction: column;
            align-items: center;
            justify-content: center;
            gap: 15px;
            z-index: 20;
        }
        .overlay-restart-btn {
            background: #ef4444;
            color: #fff;
            border: none;
            padding: 12px 24px;
            border-radius: 8px;
            font-family: inherit;
            font-weight: bold;
            font-size: 16px;
            cursor: pointer;
            box-shadow: 0 0 15px rgba(239, 68, 68, 0.4);
        }
    </style>
</head>
<body>

<div id="game-container">
    <div class="hud">
        <div>TIME: <span id="clock">12 AM</span></div>
        <div>POWER: <span id="power">100</span>%</div>
        <div>STATUS: <span id="status-text" style="color:#38bdf8;">EXPOSED</span></div>
        <button class="restart-top-btn" onclick="resetGame()">↻ RESET</button>
    </div>

    <div class="mobile-dpad" id="touch-dpad">
        <div id="dp-up" class="dpad-btn">▲</div>
        <div id="dp-left" class="dpad-btn">◄</div>
        <div id="dp-center" class="dpad-btn">●</div>
        <div id="dp-right" class="dpad-btn">►</div>
        <div id="dp-down" class="dpad-btn">▼</div>
    </div>

    <canvas id="view"></canvas>

    <div class="controls">
        <div id="run-btn" class="btn" onclick="toggleRun()">WALK</div>
        <div id="hide-btn" class="btn" onclick="toggleHide()">HIDE</div>
        <div id="flash-btn" class="btn" onclick="toggleLight()">LIGHT: OFF</div>
    </div>

    <div id="overlay">
        <h2 id="over-title" style="color:#ef4444; margin:0; text-shadow: 0 0 10px #ef4444;">YOU GOT CAUGHT</h2>
        <p id="over-desc" style="margin:0; color:#71717a;">They found your location...</p>
        <button class="overlay-restart-btn" onclick="resetGame()">TRY AGAIN</button>
    </div>
</div>

<script>
const canvas = document.getElementById('view');
const ctx = canvas.getContext('2d');
canvas.width = 360;
canvas.height = 480;

const WORLD_WIDTH = 2500;
const WORLD_HEIGHT = 2500;
const GRID_SIZE = 50;
const GRID_COLS = WORLD_WIDTH / GRID_SIZE;
const GRID_ROWS = WORLD_HEIGHT / GRID_SIZE;

let time = 0;
let power = 100;
let isRunning = false;
let lightOn = false;
let isGameOver = false;
let isHidden = false;

let juicyJeffSightTimer = 0;

const player = { x: 2420, y: 2420, radius: 10, speed: 2.5, angle: -Math.PI / 2 };

const juicyJeff = { 
    x: 200, y: 200, radius: 18, speed: 2.2, active: true, name: "JUICY JEFF",
    path: [], currentWaypoint: 0, lastPathUpdate: 0, wanderTarget: null,
    lastKnownPlayerPos: null
};

const bigWed = { 
    x: 2300, y: 200, radius: 24, speed: 2.6, active: false, name: "BIG WED",
    path: [], currentWaypoint: 0, lastPathUpdate: 0, wanderTarget: null,
    lastKnownPlayerPos: null
};

const walls = [
    { x: 0, y: 0, w: WORLD_WIDTH, h: 20 },
    { x: 0, y: WORLD_HEIGHT - 20, w: WORLD_WIDTH, h: 20 },
    { x: 0, y: 0, w: 20, h: WORLD_HEIGHT },
    { x: WORLD_WIDTH - 20, y: 0, w: 20, h: WORLD_HEIGHT },

    { x: 200, y: 20, w: 20, h: 600 },
    { x: 200, y: 620, w: 400, h: 20 },
    { x: 600, y: 200, w: 20, h: 440 },
    { x: 20, y: 400, w: 180, h: 20 },

    { x: 1200, y: 20, w: 20, h: 800 },
    { x: 1200, y: 500, w: 600, h: 20 },
    { x: 1800, y: 200, w: 20, h: 600 },
    { x: 2000, y: 400, w: 480, h: 20 },

    { x: 800, y: 800, w: 900, h: 20 },
    { x: 800, y: 800, w: 20, h: 900 },
    { x: 1700, y: 800, w: 20, h: 900 },
    { x: 1000, y: 1200, w: 500, h: 20 },
    { x: 1250, y: 1000, w: 20, h: 500 },

    { x: 200, y: 1400, w: 600, h: 20 },
    { x: 500, y: 1400, w: 20, h: 800 },
    { x: 20, y: 1800, w: 300, h: 20 },
    { x: 300, y: 1800, w: 20, h: 680 },

    { x: 1900, y: 1400, w: 20, h: 1080 },
    { x: 1400, y: 1900, w: 500, h: 20 },
    { x: 1900, y: 2200, w: 450, h: 20 },
    { x: 2150, y: 1700, w: 20, h: 500 }
];

// Grid generation for A*
const grid = Array.from({ length: GRID_ROWS }, () => Array(GRID_COLS).fill(0));

function buildNavGrid() {
    for (let r = 0; r < GRID_ROWS; r++) {
        for (let c = 0; c < GRID_COLS; c++) {
            const cellX = c * GRID_SIZE;
            const cellY = r * GRID_SIZE;
            const margin = 15;
            let isBlocked = false;
            for (let w of walls) {
                if (cellX + GRID_SIZE + margin > w.x && cellX - margin < w.x + w.w &&
                    cellY + GRID_SIZE + margin > w.y && cellY - margin < w.y + w.h) {
                    isBlocked = true;
                    break;
                }
            }
            grid[r][c] = isBlocked ? 1 : 0;
        }
    }
}
buildNavGrid();

// Raycasting to check Direct Line of Sight (LOS)
function hasLineOfSight(x1, y1, x2, y2) {
    const dx = x2 - x1;
    const dy = y2 - y1;
    const distance = Math.hypot(dx, dy);
    const steps = Math.ceil(distance / 10);
    const stepX = dx / steps;
    const stepY = dy / steps;

    let currX = x1;
    let currY = y1;

    for (let i = 0; i <= steps; i++) {
        for (let w of walls) {
            if (currX >= w.x - 5 && currX <= w.x + w.w + 5 &&
                currY >= w.y - 5 && currY <= w.y + w.h + 5) {
                return false;
            }
        }
        currX += stepX;
        currY += stepY;
    }
    return true;
}

// A* Pathfinding Algorithm
function findPath(startX, startY, targetX, targetY) {
    const startC = Math.floor(Math.max(0, Math.min(WORLD_WIDTH - 1, startX)) / GRID_SIZE);
    const startR = Math.floor(Math.max(0, Math.min(WORLD_HEIGHT - 1, startY)) / GRID_SIZE);
    const targetC = Math.floor(Math.max(0, Math.min(WORLD_WIDTH - 1, targetX)) / GRID_SIZE);
    const targetR = Math.floor(Math.max(0, Math.min(WORLD_HEIGHT - 1, targetY)) / GRID_SIZE);

    if (startC === targetC && startR === targetR) return [];

    class Node {
        constructor(r, c, g = 0, h = 0, parent = null) {
            this.r = r;
            this.c = c;
            this.g = g;
            this.h = h;
            this.f = g + h;
            this.parent = parent;
        }
    }

    const openList = [];
    const closedSet = new Set();

    const startNode = new Node(startR, startC, 0, Math.abs(startC - targetC) + Math.abs(startR - targetR));
    openList.push(startNode);

    const neighbors = [
        { r: -1, c: 0 }, { r: 1, c: 0 }, { r: 0, c: -1 }, { r: 0, c: 1 },
        { r: -1, c: -1 }, { r: -1, c: 1 }, { r: 1, c: -1 }, { r: 1, c: 1 }
    ];

    let steps = 0;
    while (openList.length > 0 && steps < 1000) {
        steps++;
        openList.sort((a, b) => a.f - b.f);
        const current = openList.shift();

        if (current.r === targetR && current.c === targetC) {
            const path = [];
            let curr = current;
            while (curr) {
                path.push({
                    x: curr.c * GRID_SIZE + GRID_SIZE / 2,
                    y: curr.r * GRID_SIZE + GRID_SIZE / 2
                });
                curr = curr.parent;
            }
            return path.reverse();
        }

        closedSet.add(`${current.r},${current.c}`);

        for (let n of neighbors) {
            const nr = current.r + n.r;
            const nc = current.c + n.c;

            if (nr < 0 || nr >= GRID_ROWS || nc < 0 || nc >= GRID_COLS) continue;
            if (grid[nr][nc] === 1) continue;
            if (closedSet.has(`${nr},${nc}`)) continue;

            const isDiagonal = n.r !== 0 && n.c !== 0;
            const dist = isDiagonal ? 1.414 : 1;
            const gScore = current.g + dist;

            let existing = openList.find(node => node.r === nr && node.c === nc);
            if (!existing) {
                const hScore = Math.hypot(nc - targetC, nr - targetR);
                openList.push(new Node(nr, nc, gScore, hScore, current));
            } else if (gScore < existing.g) {
                existing.g = gScore;
                existing.f = gScore + existing.h;
                existing.parent = current;
            }
        }
    }

    return [];
}

const hidingSpots = [
    { x: 100, y: 100, w: 50, h: 50 },
    { x: 1000, y: 100, w: 50, h: 50 },
    { x: 2350, y: 100, w: 50, h: 50 },
    { x: 100, y: 1200, w: 50, h: 50 },
    { x: 1200, y: 1300, w: 50, h: 50 },
    { x: 2350, y: 1200, w: 50, h: 50 },
    { x: 100, y: 2350, w: 50, h: 50 },
    { x: 1200, y: 2350, w: 50, h: 50 },
    { x: 2400, y: 2400, w: 50, h: 50 }
];

const dustParticles = [];
for (let i = 0; i < 150; i++) {
    dustParticles.push({
        x: Math.random() * WORLD_WIDTH,
        y: Math.random() * WORLD_HEIGHT,
        size: Math.random() * 1.5 + 0.5,
        alpha: Math.random() * 0.4 + 0.1,
        speedY: Math.random() * 0.2 - 0.1,
        speedX: Math.random() * 0.2 - 0.1
    });
}

const keys = { up: false, down: false, left: false, right: false };

const dpad = document.getElementById('touch-dpad');
function handleDpadTouch(e) {
    e.preventDefault();
    if (isHidden) return;
    keys.up = keys.down = keys.left = keys.right = false;

    if (e.type === 'touchend' && e.touches.length === 0) return;

    const rect = dpad.getBoundingClientRect();
    const touch = e.touches[0] || e;
    const x = touch.clientX - rect.left - rect.width / 2;
    const y = touch.clientY - rect.top - rect.height / 2;

    const threshold = 10;
    if (Math.abs(x) > threshold) {
        if (x > 0) keys.right = true;
        if (x < 0) keys.left = true;
    }
    if (Math.abs(y) > threshold) {
        if (y > 0) keys.down = true;
        if (y < 0) keys.up = true;
    }

    document.getElementById('dp-up').classList.toggle('active', keys.up);
    document.getElementById('dp-down').classList.toggle('active', keys.down);
    document.getElementById('dp-left').classList.toggle('active', keys.left);
    document.getElementById('dp-right').classList.toggle('active', keys.right);
}

['touchstart', 'touchmove', 'touchend', 'mousedown', 'mousemove'].forEach(evt => {
    dpad.addEventListener(evt, (e) => {
        if (evt === 'mousedown' || evt === 'mousemove') {
            if (e.buttons !== 1) return;
        }
        handleDpadTouch(e);
    });
});

window.addEventListener('mouseup', () => {
    keys.up = keys.down = keys.left = keys.right = false;
    document.querySelectorAll('.dpad-btn').forEach(b => b.classList.remove('active'));
});

window.addEventListener('keydown', (e) => {
    if (isHidden) return;
    if (e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W') keys.up = true;
    if (e.key === 'ArrowDown' || e.key === 's' || e.key === 'S') keys.down = true;
    if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') keys.left = true;
    if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') keys.right = true;
});
window.addEventListener('keyup', (e) => {
    if (e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W') keys.up = false;
    if (e.key === 'ArrowDown' || e.key === 's' || e.key === 'S') keys.down = false;
    if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') keys.left = false;
    if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') keys.right = false;
});

function toggleRun() {
    if (isGameOver || isHidden) return;
    isRunning = !isRunning;
    player.speed = isRunning ? 4.5 : 2.5;
    const btn = document.getElementById('run-btn');
    btn.textContent = isRunning ? 'RUNNING' : 'WALK';
    btn.classList.toggle('active', isRunning);
}

function toggleLight() {
    if (isGameOver || isHidden) return;
    lightOn = !lightOn;
    const btn = document.getElementById('flash-btn');
    btn.textContent = `LIGHT: ${lightOn ? 'ON' : 'OFF'}`;
}

function toggleHide() {
    if (isGameOver) return;
    const statusText = document.getElementById('status-text');
    const hideBtn = document.getElementById('hide-btn');

    if (isHidden) {
        isHidden = false;
        hideBtn.classList.remove('hide-active');
        statusText.textContent = 'EXPOSED';
        statusText.style.color = '#38bdf8';
        return;
    }

    const currentSpot = hidingSpots.find(spot => 
        player.x >= spot.x && player.x <= spot.x + spot.w &&
        player.y >= spot.y && player.y <= spot.y + spot.h
    );

    if (currentSpot) {
        isHidden = true;
        lightOn = false;
        isRunning = false;
        keys.up = keys.down = keys.left = keys.right = false;
        document.getElementById('flash-btn').textContent = 'LIGHT: OFF';
        document.getElementById('run-btn').textContent = 'WALK';
        document.getElementById('run-btn').classList.remove('active');
        hideBtn.classList.add('hide-active');
        statusText.textContent = 'HIDDEN';
        statusText.style.color = '#22c55e';
    }
}

setInterval(() => {
    if (isGameOver) return;
    
    let drain = 0.03;
    if (lightOn) drain += 0.15;
    if (isRunning) drain += 0.08;
    power = Math.max(0, power - drain);
    document.getElementById('power').textContent = Math.round(power);
    
    if (power === 0) lightOn = false;

    let isJeffVisible = false;
    if (lightOn && juicyJeff.active) {
        const dx = juicyJeff.x - player.x;
        const dy = juicyJeff.y - player.y;
        const dist = Math.hypot(dx, dy);
        const lightRange = 220;
        const fovAngle = Math.PI / 2.5;

        if (dist <= lightRange) {
            let angleToJeff = Math.atan2(dy, dx);
            let angleDiff = angleToJeff - player.angle;
            
            while (angleDiff > Math.PI) angleDiff -= Math.PI * 2;
            while (angleDiff < -Math.PI) angleDiff += Math.PI * 2;

            if (Math.abs(angleDiff) <= fovAngle / 2) {
                isJeffVisible = true;
            }
        }
    }

    if (isJeffVisible) {
        juicyJeffSightTimer = 0;
    } else if (juicyJeff.active) {
        juicyJeffSightTimer++;
        if (juicyJeffSightTimer >= 30) {
            juicyJeffSightTimer = 0;
            const spawnDistance = 180;
            const backAngle = player.angle + Math.PI;
            
            let spawnX = player.x + Math.cos(backAngle) * spawnDistance;
            let spawnY = player.y + Math.sin(backAngle) * spawnDistance;

            spawnX = Math.max(50, Math.min(WORLD_WIDTH - 50, spawnX));
            spawnY = Math.max(50, Math.min(WORLD_HEIGHT - 50, spawnY));

            juicyJeff.x = spawnX;
            juicyJeff.y = spawnY;
            juicyJeff.path = [];
        }
    }
}, 1000);

setInterval(() => {
    if (isGameOver) return;
    time = (time + 1) % 7;
    const hours = ["12 AM", "1 AM", "2 AM", "3 AM", "4 AM", "5 AM", "6 AM"];
    document.getElementById('clock').textContent = hours[time];

    if (time === 3 && !bigWed.active) {
        bigWed.active = true;
    }

    if (time === 6) endGame(true, null);
}, 180000);

function checkWallCollision(newX, newY, radius) {
    for (let w of walls) {
        if (newX + radius > w.x && newX - radius < w.x + w.w &&
            newY + radius > w.y && newY - radius < w.y + w.h) {
            return true;
        }
    }
    return false;
}

function getRandomUnblockedLocation() {
    let x, y, c, r;
    do {
        x = Math.random() * (WORLD_WIDTH - 200) + 100;
        y = Math.random() * (WORLD_HEIGHT - 200) + 100;
        c = Math.floor(x / GRID_SIZE);
        r = Math.floor(y / GRID_SIZE);
    } while (grid[r][c] === 1);
    return { x, y };
}

function updateEntityAI(entity) {
    if (!entity.active) return;

    const now = Date.now();
    const distToPlayer = Math.hypot(player.x - entity.x, player.y - entity.y);
    const canSee = !isHidden && hasLineOfSight(entity.x, entity.y, player.x, player.y);

    let moveSpeed = entity.speed;

    if (canSee) {
        // Direct Line of Sight Pursue: Smooth direct tracking without grid limits
        entity.lastKnownPlayerPos = { x: player.x, y: player.y };
        entity.path = []; // Clear grid path to use direct vectoring
        
        // Speed boost when actively chasing visible target
        moveSpeed *= 1.35;

        const dx = player.x - entity.x;
        const dy = player.y - entity.y;
        const angle = Math.atan2(dy, dx);

        const nextX = entity.x + Math.cos(angle) * moveSpeed;
        const nextY = entity.y + Math.sin(angle) * moveSpeed;

        if (!checkWallCollision(nextX, entity.y, entity.radius)) entity.x = nextX;
        if (!checkWallCollision(entity.x, nextY, entity.radius)) entity.y = nextY;
    } 
    else {
        // Lost LOS or Player is Hidden: Navigate to last known position or wander via A*
        if (!isHidden && entity.lastKnownPlayerPos) {
            const distToLastPos = Math.hypot(entity.lastKnownPlayerPos.x - entity.x, entity.lastKnownPlayerPos.y - entity.y);
            if (distToLastPos < 20) {
                entity.lastKnownPlayerPos = null; // Reached last known spot, start wandering
            } else if (now - entity.lastPathUpdate > 400) {
                entity.path = findPath(entity.x, entity.y, entity.lastKnownPlayerPos.x, entity.lastKnownPlayerPos.y);
                entity.currentWaypoint = 0;
                entity.lastPathUpdate = now;
            }
        } 
        else if (!entity.wanderTarget || Math.hypot(entity.wanderTarget.x - entity.x, entity.wanderTarget.y - entity.y) < 30) {
            entity.wanderTarget = getRandomUnblockedLocation();
            entity.path = findPath(entity.x, entity.y, entity.wanderTarget.x, entity.wanderTarget.y);
            entity.currentWaypoint = 0;
        }

        // Follow A* Path Waypoints
        if (entity.path && entity.path.length > 0 && entity.currentWaypoint < entity.path.length) {
            const target = entity.path[entity.currentWaypoint];
            const dx = target.x - entity.x;
            const dy = target.y - entity.y;
            const distToWaypoint = Math.hypot(dx, dy);

            if (distToWaypoint < moveSpeed) {
                entity.x = target.x;
                entity.y = target.y;
                entity.currentWaypoint++;
            } else {
                const nextX = entity.x + (dx / distToWaypoint) * moveSpeed;
                const nextY = entity.y + (dy / distToWaypoint) * moveSpeed;

                if (!checkWallCollision(nextX, entity.y, entity.radius)) entity.x = nextX;
                if (!checkWallCollision(entity.x, nextY, entity.radius)) entity.y = nextY;
            }
        }
    }

    if (!isHidden && distToPlayer < player.radius + entity.radius) {
        endGame(false, entity.name);
    }
}

function update() {
    if (isGameOver) return;

    if (!isHidden) {
        let moveX = 0;
        let moveY = 0;
        if (keys.up) moveY -= 1;
        if (keys.down) moveY += 1;
        if (keys.left) moveX -= 1;
        if (keys.right) moveX += 1;

        if (moveX !== 0 || moveY !== 0) {
            player.angle = Math.atan2(moveY, moveX);
            if (moveX !== 0 && moveY !== 0) {
                moveX *= 0.7071;
                moveY *= 0.7071;
            }

            const nextX = player.x + moveX * player.speed;
            const nextY = player.y + moveY * player.speed;

            if (!checkWallCollision(nextX, player.y, player.radius)) player.x = nextX;
            if (!checkWallCollision(player.x, nextY, player.radius)) player.y = nextY;
        }
    }

    dustParticles.forEach(p => {
        p.x += p.speedX;
        p.y += p.speedY;
        if (p.x < 0) p.x = WORLD_WIDTH;
        if (p.x > WORLD_WIDTH) p.x = 0;
        if (p.y < 0) p.y = WORLD_HEIGHT;
        if (p.y > WORLD_HEIGHT) p.y = 0;
    });

    updateEntityAI(juicyJeff);
    updateEntityAI(bigWed);
}

function drawEntity(entity, lightRange, mainColor, eyeColor) {
    if (!entity.active) return;

    const dist = Math.hypot(player.x - entity.x, player.y - entity.y);
    if (dist < lightRange) {
        ctx.fillStyle = mainColor;
        ctx.beginPath();
        ctx.arc(entity.x, entity.y, entity.radius, 0, Math.PI * 2);
        ctx.fill();

        ctx.fillStyle = eyeColor;
        ctx.beginPath();
        ctx.arc(entity.x - 6, entity.y - 3, 3, 0, Math.PI * 2);
        ctx.arc(entity.x + 6, entity.y - 3, 3, 0, Math.PI * 2);
        ctx.fill();
    }
}

function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    ctx.fillStyle = '#000000';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    const camX = Math.max(0, Math.min(WORLD_WIDTH - canvas.width, player.x - canvas.width / 2));
    const camY = Math.max(0, Math.min(WORLD_HEIGHT - canvas.height, player.y - canvas.height / 2));

    ctx.save();
    ctx.translate(-camX, -camY);

    if (lightOn) {
        ctx.fillStyle = '#030303';
        ctx.fillRect(0, 0, WORLD_WIDTH, WORLD_HEIGHT);

        ctx.fillStyle = '#27272a';
        ctx.strokeStyle = '#3f3f46';
        walls.forEach(w => {
            ctx.fillRect(w.x, w.y, w.w, w.h);
            ctx.strokeRect(w.x, w.y, w.w, w.h);
        });

        hidingSpots.forEach(s => {
            ctx.fillStyle = '#14532d';
            ctx.strokeStyle = '#22c55e';
            ctx.fillRect(s.x, s.y, s.w, s.h);
            ctx.strokeRect(s.x, s.y, s.w, s.h);
        });

        ctx.save();
        ctx.beginPath();
        ctx.moveTo(player.x, player.y);
        const fovAngle = Math.PI / 2.5;
        const lightRange = 220;
        ctx.arc(player.x, player.y, lightRange, player.angle - fovAngle / 2, player.angle + fovAngle / 2);
        ctx.closePath();

        const grad = ctx.createRadialGradient(player.x, player.y, 5, player.x, player.y, lightRange);
        grad.addColorStop(0, 'rgba(254, 240, 138, 0.5)');
        grad.addColorStop(0.7, 'rgba(254, 240, 138, 0.12)');
        grad.addColorStop(1, 'rgba(0, 0, 0, 0.98)');
        ctx.fillStyle = grad;
        ctx.fill();
        ctx.restore();

        dustParticles.forEach(p => {
            const d = Math.hypot(player.x - p.x, player.y - p.y);
            if (d < lightRange) {
                ctx.fillStyle = `rgba(255, 255, 255, ${p.alpha})`;
                ctx.beginPath();
                ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
                ctx.fill();
            }
        });

        drawEntity(juicyJeff, lightRange, '#991b1b', '#ff0000');
        drawEntity(bigWed, lightRange, '#581c87', '#a855f7');
    }

    if (!isHidden) {
        ctx.fillStyle = '#38bdf8';
        ctx.beginPath();
        ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
        ctx.fill();
    }

    ctx.restore();

    requestAnimationFrame(draw);
}

function endGame(won, caughtBy) {
    isGameOver = true;
    const overlay = document.getElementById('overlay');
    const title = document.getElementById('over-title');
    const desc = document.getElementById('over-desc');

    overlay.style.display = 'flex';
    if (won) {
        title.textContent = "6 AM - SURVIVED";
        title.style.color = "#4ade80";
        title.style.textShadow = "0 0 10px #4ade80";
        desc.textContent = "You survived Juicy Jeff and Big Wed!";
    } else {
        title.textContent = `${caughtBy} GOT YOU`;
        title.style.color = "#ef4444";
        title.style.textShadow = "0 0 10px #ef4444";
        desc.textContent = "You were caught in the darkness.";
    }
}

function resetGame() {
    player.x = 2420; 
    player.y = 2420;
    player.angle = -Math.PI / 2;
    
    juicyJeff.x = 200; 
    juicyJeff.y = 200;
    juicyJeff.active = true;
    juicyJeff.path = [];
    juicyJeff.lastKnownPlayerPos = null;

    bigWed.x = 2300;
    bigWed.y = 200;
    bigWed.active = false;
    bigWed.path = [];
    bigWed.lastKnownPlayerPos = null;

    juicyJeffSightTimer = 0;

    power = 100;
    time = 0;
    isRunning = false;
    lightOn = false;
    isGameOver = false;
    isHidden = false;

    document.getElementById('power').textContent = '100';
    document.getElementById('clock').textContent = '12 AM';
    
    const statusText = document.getElementById('status-text');
    statusText.textContent = 'EXPOSED';
    statusText.style.color = '#38bdf8';

    const runBtn = document.getElementById('run-btn');
    runBtn.textContent = 'WALK';
    runBtn.classList.remove('active');
    
    document.getElementById('hide-btn').classList.remove('hide-active');
    document.getElementById('flash-btn').textContent = 'LIGHT: OFF';
    document.getElementById('overlay').style.display = 'none';
}

setInterval(update, 1000 / 60);
draw();
</script>
</body>
</html>
