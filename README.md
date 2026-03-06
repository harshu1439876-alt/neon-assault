# neon-assault
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Galaxy Assault: Neon Supernova</title>
    <style>
        body { margin: 0; background: #000105; color: white; font-family: 'Segoe UI', Arial; overflow: hidden; user-select: none; }
        canvas { display: block; }
        #menu, #gameOver, #cardScreen {
            position: absolute; inset: 0; display: flex; align-items: center; justify-content: center;
            flex-direction: column; background: rgba(0,0,0,0.9); z-index: 10; text-align: center;
        }
        #ui { position: absolute; top: 15px; left: 15px; z-index: 5; width: 220px; pointer-events: none; }
        .hb-bg { width: 100%; height: 10px; background: #111; border: 2px solid #0ff; border-radius: 5px; margin-top: 5px; overflow: hidden; box-shadow: 0 0 10px #0ff; }
        #hb-fill { width: 100%; height: 100%; background: #0ff; transition: width 0.3s; }
        #cardContainer { display: flex; gap: 20px; margin-top: 30px; }
        .card {
            width: 170px; height: 250px; background: #050505; border: 3px solid #f0f; border-radius: 12px;
            padding: 15px; cursor: pointer; transition: 0.2s; box-shadow: 0 0 15px #f0f;
        }
        .card:hover { transform: scale(1.05); box-shadow: 0 0 25px #0ff; border-color: #0ff; }
        button { padding: 12px 30px; font-size: 18px; background: #0ff; border: none; cursor: pointer; font-weight: bold; border-radius: 5px; color: black; box-shadow: 0 0 15px #0ff; }
        #lvlMsg { position: absolute; top: 40%; left: 50%; transform: translate(-50%, -50%); font-size: 50px; color: #0ff; font-weight: bold; display: none; z-index: 9; text-shadow: 0 0 20px #0ff; }
    </style>
</head>
<body>

<div id="ui" style="display:none">
    WAVE: <span id="ld">1</span> | SCORE: <span id="sd">0</span>
    <div class="hb-bg"><div id="hb-fill"></div></div>
</div>

<div id="lvlMsg">WAVE START</div>

<div id="menu">
    <h1 style="color:#0ff; text-shadow: 0 0 20px #0ff;">NEON ASSAULT</h1>
    <button onclick="startGame()">INITIALIZE ENGINE</button>
</div>

<div id="cardScreen" style="display:none">
    <h2 style="color:#f0f;">EVOLUTION SELECT</h2>
    <div id="cardContainer"></div>
</div>

<div id="gameOver" style="display:none">
    <h1 style="color:#f00;">HULL BREACHED</h1>
    <button onclick="resetGame()">RE-INITIALIZE</button>
</div>

<canvas id="g"></canvas>

<script>
const cvs = document.getElementById("g"), ctx = cvs.getContext("2d");
cvs.width = window.innerWidth; cvs.height = window.innerHeight;

const audio = new (window.AudioContext || window.webkitAudioContext)();
let bgmInterval;

function playBGM() {
    if(bgmInterval) clearInterval(bgmInterval);
    let step = 0;
    const m = [82, 110, 82, 123, 82, 146, 123, 110];
    bgmInterval = setInterval(() => {
        if (!gameActive || paused) return;
        const o = audio.createOscillator(), g = audio.createGain();
        o.type = 'sawtooth'; o.frequency.setValueAtTime(m[step % 8], audio.currentTime);
        g.gain.setValueAtTime(0.02, audio.currentTime); g.gain.exponentialRampToValueAtTime(0.001, audio.currentTime + 0.2);
        o.connect(g); g.connect(audio.destination); o.start(); o.stop(audio.currentTime + 0.2);
        step++;
    }, 220); 
}

let gameActive = false, paused = false, score = 0, level = 1, nextLevelScore = 1500;
let bullets = [], eBullets = [], enemies = [], stars = [], nebulas = [], comets = [], orbs = [], particles = [];
let shield = 0, keys = {}, laserTimer = 0;
let stats = { fireRate: 11, bSpeed: 14, dmg: 2, maxH: 150 };
let player = { x: cvs.width/2, y: cvs.height-100, h: 150, cd: 0, dead: false, exploding: false };

window.onkeydown = e => keys[e.key] = true;
window.onkeyup = e => keys[e.key] = false;

function spawnParticles(x, y, color, count = 8, speed = 8) {
    for(let i=0; i<count; i++) {
        particles.push({ x, y, vx: (Math.random()-0.5)*speed, vy: (Math.random()-0.5)*speed, life: 1.0, color });
    }
}

function startGame() {
    if(audio.state === 'suspended') audio.resume();
    document.getElementById("menu").style.display = "none";
    document.getElementById("ui").style.display = "block";
    gameActive = true;
    for(let i=0; i<100; i++) stars.push({x: Math.random()*cvs.width, y: Math.random()*cvs.height, s: Math.random()*2+0.5});
    for(let i=0; i<6; i++) spawnNebula();
    playBGM(); announceLevel();
}

function spawnNebula() {
    const cols = ["rgba(150,0,255,0.08)", "rgba(0,200,255,0.08)", "rgba(255,0,100,0.05)"];
    nebulas.push({ 
        x: Math.random()*cvs.width, 
        y: Math.random()*cvs.height - cvs.height, 
        r: 200+Math.random()*300, 
        v: 0.2+Math.random()*0.4, 
        c: cols[Math.floor(Math.random()*cols.length)],
        angle: Math.random() * Math.PI * 2,
        spin: (Math.random() - 0.5) * 0.01
    });
}

function spawnComet() {
    comets.push({ x: Math.random()<0.5?-50:cvs.width+50, y: Math.random()*(cvs.height/2), vx: (4+Math.random()*4)*(Math.random()<0.5?1:-1), vy: 2, h: 6, r: 20 });
}

function resetGame() {
    score = 0; level = 1; nextLevelScore = 1500; bullets = []; eBullets = []; enemies = []; comets = []; orbs = []; particles = []; shield = 0; laserTimer = 0;
    player.h = 150; player.dead = false; player.exploding = false; player.x = cvs.width/2; player.y = cvs.height-100;
    document.getElementById("gameOver").style.display = "none";
    paused = false; playBGM(); announceLevel();
}

function announceLevel() {
    const msg = document.getElementById("lvlMsg");
    msg.innerText = "WAVE " + level; msg.style.display = "block";
    setTimeout(() => { 
        msg.style.display = "none"; 
        if(level > 1 && level % 2 === 0) showCards();
    }, 1800);
}

function showCards() {
    paused = true; const container = document.getElementById("cardContainer"); container.innerHTML = "";
    const options = [ { id: 'laser', n: 'MEGA LASER', d: 'Pierce everything', c: '#0ff' }, { id: 'repair', n: 'REPAIR NANO', d: 'Full Heal', c: '#4f4' } ];
    options.forEach(o => {
        const d = document.createElement("div"); d.className = "card"; d.style.borderColor = o.c;
        d.innerHTML = `<h3>${o.n}</h3><p>${o.d}</p>`;
        d.onclick = () => { if(o.id === 'laser') laserTimer = 480; if(o.id === 'repair') player.h = stats.maxH; document.getElementById("cardScreen").style.display = "none"; paused = false; };
        container.appendChild(d);
    });
    document.getElementById("cardScreen").style.display = "flex";
}

function drawRocket(x, y, color, size, isEnemy = true) {
    ctx.save();
    ctx.translate(x, y);
    if (!isEnemy) {
        const fH = size * (0.8 + Math.random() * 0.5);
        ctx.shadowBlur = 15; ctx.shadowColor = "#f90";
        ctx.fillStyle = "#f40";
        ctx.beginPath(); ctx.moveTo(-size/4, size/2); ctx.lineTo(0, size/2 + fH); ctx.lineTo(size/4, size/2); ctx.fill();
        ctx.fillStyle = "#ff0";
        ctx.beginPath(); ctx.moveTo(-size/6, size/2); ctx.lineTo(0, size/2 + fH/1.5); ctx.lineTo(size/6, size/2); ctx.fill();
    } else { ctx.rotate(Math.PI); }
    ctx.shadowBlur = 15; ctx.shadowColor = color; ctx.fillStyle = color;
    ctx.beginPath(); ctx.moveTo(0, -size); ctx.lineTo(size/2, size/2); ctx.lineTo(-size/2, size/2); ctx.closePath(); ctx.fill();
    ctx.beginPath(); ctx.moveTo(-size/2, 0); ctx.lineTo(-size, size/2); ctx.lineTo(-size/2, size/2); ctx.fill();
    ctx.beginPath(); ctx.moveTo(size/2, 0); ctx.lineTo(size, size/2); ctx.lineTo(size/2, size/2); ctx.fill();
    ctx.fillStyle = "#fff"; ctx.beginPath(); ctx.arc(0, -size/4, size/5, 0, Math.PI*2); ctx.fill();
    ctx.restore();
}

function update() {
    if(!gameActive || paused || player.dead) return;
    
    if(!player.exploding) {
        if(keys.ArrowLeft && player.x > 30) player.x -= 9;
        if(keys.ArrowRight && player.x < cvs.width-30) player.x += 9;
        if(keys.ArrowUp && player.y > 30) player.y -= 9;
        if(keys.ArrowDown && player.y < cvs.height-30) player.y += 9;
        if(shield > 0) shield--;
        if(laserTimer > 0) laserTimer--;

        if(keys[" "] && player.cd <= 0 && laserTimer <= 0) {
            bullets.push({x:player.x, y:player.y-30});
            player.cd = stats.fireRate;
        }
        player.cd--;
    }

    if(enemies.length < (3 + level) && Math.random() < 0.02) {
        let type = (level >= 3 && Math.random() < 0.2) ? 'stalker' : (level >= 2 && Math.random() < 0.15 ? 'heavy' : 'normal');
        enemies.push({x: Math.random()*cvs.width, y: -100, h: (type==='heavy'?8:4)+level, ls: Date.now(), type: type});
    }
    if(Math.random() < 0.002) spawnComet();

    bullets.forEach((b, i) => { b.y -= stats.bSpeed; if(b.y < -50) bullets.splice(i, 1); });

    enemies.forEach((e, i) => {
        e.y += (e.type === 'heavy' ? 1.5 : 3);
        if(e.y > cvs.height + 50) { enemies.splice(i, 1); return; }

        if(e.type === 'stalker') e.x += (player.x - e.x) * 0.025;
        let shootDelay = e.type === 'heavy' ? 900 : 1300; 
        if(Date.now() - e.ls > shootDelay) {
            let angle = Math.atan2(player.y - e.y, player.x - e.x);
            eBullets.push({ x: e.x, y: e.y, vx: Math.cos(angle) * 5.5, vy: Math.sin(angle) * 5.5, size: e.type === 'heavy' ? 14 : 6, dmg: e.type === 'heavy' ? 30 : 15 });
            e.ls = Date.now();
        }
        if(laserTimer > 0 && keys[" "] && Math.abs(e.x - player.x) < 25 && e.y < player.y) { e.h -= 0.5; spawnParticles(e.x, e.y, "#0ff"); }
        bullets.forEach((b, bi) => { if(Math.hypot(e.x-b.x, e.y-b.y) < 30) { e.h -= stats.dmg; bullets.splice(bi, 1); } });
        if(e.h <= 0) { 
            let col = e.type==='stalker'?"#0f6":(e.type==='heavy'?"#f90":"#f06");
            spawnParticles(e.x, e.y, col);
            score += 100; enemies.splice(i,1);
            if(score >= nextLevelScore) { level++; nextLevelScore += 2000; announceLevel(); }
        }
    });

    eBullets.forEach((eb, i) => {
        eb.x += eb.vx; eb.y += eb.vy;
        if(!player.exploding && Math.hypot(player.x-eb.x, player.y-eb.y) < (eb.size + 10)) { 
            if(shield <= 0) player.h -= eb.dmg;
            eBullets.splice(i, 1); 
        }
        if(eb.y > cvs.height || eb.x < 0 || eb.x > cvs.width) eBullets.splice(i,1);
    });

    particles.forEach((p, i) => { p.x += p.vx; p.y += p.vy; p.life -= 0.02; if(p.life <= 0) particles.splice(i, 1); });
    
    comets.forEach((c, i) => {
        c.x += c.vx; c.y += c.vy;
        if(laserTimer > 0 && keys[" "] && Math.abs(c.x - player.x) < 25 && c.y < player.y) c.h -= 0.2;
        bullets.forEach((b, bi) => { if(Math.hypot(c.x-b.x, c.y-b.y) < 30) { c.h -= stats.dmg; bullets.splice(bi,1); } });
        if(c.h <= 0) { spawnParticles(c.x, c.y, "#fff"); orbs.push({x:c.x, y:c.y}); comets.splice(i,1); }
    });

    orbs.forEach((o, i) => { o.y += 2; if(Math.hypot(player.x-o.x, player.y-o.y) < 40) { shield = 400; orbs.splice(i,1); } });

    document.getElementById("hb-fill").style.width = Math.max(0, (player.h / stats.maxH)*100) + "%";
    document.getElementById("sd").innerText = score; document.getElementById("ld").innerText = level;

    // Trigger Player Explosion
    if(player.h <= 0 && !player.exploding) { 
        player.exploding = true;
        spawnParticles(player.x, player.y, "#0ff", 40, 15); // Large neon pieces
        setTimeout(() => {
            player.dead = true;
            document.getElementById("gameOver").style.display = "flex";
        }, 1200);
    }
}

function draw() {
    ctx.fillStyle = "#000108"; ctx.fillRect(0,0,cvs.width,cvs.height);
    nebulas.forEach(n => {
        n.y += n.v; n.angle += n.spin;
        ctx.save(); ctx.translate(n.x, n.y); ctx.rotate(n.angle);
        let g = ctx.createRadialGradient(0, 0, 0, 0, 0, n.r);
        g.addColorStop(0, n.c); g.addColorStop(0.5, n.c.replace("0.08", "0.03")); g.addColorStop(1, 'transparent');
        ctx.fillStyle = g; ctx.beginPath(); ctx.ellipse(0, 0, n.r, n.r * 0.7, 0, 0, Math.PI * 2); ctx.fill(); ctx.restore();
        if(n.y - n.r > cvs.height) { n.y = -n.r; n.x = Math.random() * cvs.width; }
    });
    stars.forEach(s => { s.y += s.s; if(s.y > cvs.height) s.y = 0; ctx.fillStyle="#fff"; ctx.fillRect(s.x, s.y, 1, 1); });

    if(gameActive) {
        if(!player.exploding) {
            if(shield > 0) { ctx.strokeStyle = "#0ff"; ctx.beginPath(); ctx.arc(player.x, player.y, 45, 0, Math.PI*2); ctx.stroke(); }
            drawRocket(player.x, player.y, "#0ff", 25, false);
            if(laserTimer > 0 && keys[" "]) {
                ctx.shadowBlur = 20; ctx.shadowColor = "#0ff";
                ctx.fillStyle = "rgba(0, 255, 255, 0.3)"; ctx.fillRect(player.x - 20, 0, 40, player.y - 20);
                ctx.fillStyle = "#fff"; ctx.fillRect(player.x - 5, 0, 10, player.y - 20);
                ctx.shadowBlur = 0;
            }
        }
        enemies.forEach(e => {
            let col = e.type==='stalker'?"#0f6": (e.type==='heavy'?"#f90":"#f06");
            drawRocket(e.x, e.y, col, e.type==='heavy'?35:20, true);
        });
        particles.forEach(p => { ctx.globalAlpha = p.life; ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 4, 4); });
        ctx.globalAlpha = 1.0;
        bullets.forEach(b => { ctx.shadowBlur = 10; ctx.shadowColor = "#0ff"; ctx.fillStyle = "#fff"; ctx.fillRect(b.x-2, b.y, 4, 15); ctx.shadowBlur = 0; });
        eBullets.forEach(eb => { 
            ctx.shadowBlur = 15; ctx.shadowColor = "#f0f"; ctx.fillStyle = "#f0f"; 
            ctx.beginPath(); ctx.arc(eb.x, eb.y, eb.size, 0, Math.PI*2); ctx.fill(); ctx.shadowBlur = 0; 
        });
        orbs.forEach(o => { ctx.fillStyle="#0ff"; ctx.beginPath(); ctx.arc(o.x, o.y, 10, 0, Math.PI*2); ctx.fill(); });
    }
    requestAnimationFrame(() => { update(); draw(); });
}
draw();
</script>
</body>
</html>
