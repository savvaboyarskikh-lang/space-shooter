<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Space Shooter ULTRA</title>
<style>
    body {
        margin: 0;
        overflow: hidden;
        background: black;
        color: white;
        font-family: Arial;
    }
</style>
</head>
<body>

<canvas id="game"></canvas>

<script>
const canvas = document.getElementById("game");
const ctx = canvas.getContext("2d");

canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

let state = "menu";
let player, enemies, bullets, particles, stars, boss;
let score, hp, weaponLevel, bestScore = 0;

// 🔊 звуки
let shootSound = new Audio("https://assets.mixkit.co/sfx/preview/mixkit-game-laser-shoot-1681.mp3");
let boomSound = new Audio("https://assets.mixkit.co/sfx/preview/mixkit-explosion-1688.mp3");
let music = new Audio("https://assets.mixkit.co/music/preview/mixkit-arcade-game-opener-222.mp3");
music.loop = true;

// ⭐ звезды
stars = [];
for (let i = 0; i < 100; i++) {
    stars.push({
        x: Math.random()*canvas.width,
        y: Math.random()*canvas.height,
        speed: Math.random()*2+1
    });
}

// загрузка рекорда
if (localStorage.getItem("bestScore")) {
    bestScore = parseInt(localStorage.getItem("bestScore"));
}

function init() {
    player = { x: canvas.width/2, y: canvas.height-100 };

    enemies = [];
    bullets = [];
    particles = [];
    boss = null;

    score = 0;
    hp = 3;
    weaponLevel = 1;
}

// 🎮 мышка
canvas.addEventListener("mousemove", e => {
    player.x = e.clientX;
});

// клик — стрельба + старт
canvas.addEventListener("click", () => {
    if (state === "menu") {
        state = "game";
        init();
        music.play();
    } else if (state === "game") {
        shoot();
    }
});

// клавиши
document.addEventListener("keydown", e => {
    if (state === "gameover" && e.key === "r") {
        state = "menu";
    }
});

// стрельба
function shoot() {
    shootSound.currentTime = 0;
    shootSound.play();

    if (weaponLevel === 1) {
        bullets.push({ x: player.x, y: player.y, dx: 0 });
    } else if (weaponLevel === 2) {
        bullets.push({ x: player.x-10, y: player.y, dx: -1 });
        bullets.push({ x: player.x+10, y: player.y, dx: 1 });
    } else {
        bullets.push({ x: player.x, y: player.y, dx: 0 });
        bullets.push({ x: player.x-15, y: player.y, dx: -1 });
        bullets.push({ x: player.x+15, y: player.y, dx: 1 });
    }
}

// враги
function spawnEnemy() {
    if (state !== "game" || boss) return;

    enemies.push({
        x: Math.random()*canvas.width,
        y: -20,
        speed: 2+Math.random()*2,
        hp: 2
    });
}

// босс
function spawnBoss() {
    boss = { x: canvas.width/2, y: 100, hp: 60, dir: 1 };
}

// взрыв
function explosion(x,y){
    boomSound.currentTime = 0;
    boomSound.play();

    for(let i=0;i<15;i++){
        particles.push({
            x,y,
            dx:(Math.random()-0.5)*6,
            dy:(Math.random()-0.5)*6,
            life:40
        });
    }
}

function hit(a,b){
    return Math.abs(a.x-b.x)<30 && Math.abs(a.y-b.y)<30;
}

function update(){
    if(state !== "game") return;

    stars.forEach(s=>{
        s.y+=s.speed;
        if(s.y>canvas.height){
            s.y=0;
            s.x=Math.random()*canvas.width;
        }
    });

    bullets.forEach(b=>{
        b.y-=10;
        b.x+=b.dx;
    });

    enemies.forEach(e=>{
        e.y+=e.speed;

        if(hit(player,e)){
            hp--;
            explosion(e.x,e.y);
            e.y=9999;
            if(hp<=0){
                state="gameover";
                saveScore();
            }
        }
    });

    if(score>=300 && !boss) spawnBoss();

    if(boss){
        boss.x+=boss.dir*3;
        if(boss.x<50||boss.x>canvas.width-50) boss.dir*=-1;

        if(hit(player,boss)){
            hp--;
            if(hp<=0){
                state="gameover";
                saveScore();
            }
        }
    }

    bullets.forEach((b,bi)=>{
        enemies.forEach((e,ei)=>{
            if(hit(b,e)){
                e.hp--;
                bullets.splice(bi,1);

                if(e.hp<=0){
                    explosion(e.x,e.y);
                    enemies.splice(ei,1);
                    score+=20;

                    if(score % 100 === 0 && weaponLevel < 3){
                        weaponLevel++;
                    }
                }
            }
        });

        if(boss && hit(b,boss)){
            boss.hp--;
            bullets.splice(bi,1);

            if(boss.hp<=0){
                explosion(boss.x,boss.y);
                boss=null;
                score+=200;
            }
        }
    });

    bullets = bullets.filter(b=>b.y>0);
    enemies = enemies.filter(e=>e.y<canvas.height);
    particles = particles.filter(p=>p.life-- >0);
}

// 💾 сохранение рекорда
function saveScore() {
    if (score > bestScore) {
        bestScore = score;
        localStorage.setItem("bestScore", bestScore);
    }
}

// рисование
function draw(){
    ctx.clearRect(0,0,canvas.width,canvas.height);

    ctx.fillStyle="white";
    stars.forEach(s=>ctx.fillRect(s.x,s.y,2,2));

    if(state==="menu"){
        ctx.font="40px Arial";
        ctx.fillText("SPACE SHOOTER",canvas.width/2-180,canvas.height/2-20);
        ctx.font="20px Arial";
        ctx.fillText("Click to start",canvas.width/2-70,canvas.height/2+20);
        ctx.fillText("Best: "+bestScore,canvas.width/2-50,canvas.height/2+50);
        return;
    }

    // игрок
    ctx.fillStyle="cyan";
    ctx.beginPath();
    ctx.moveTo(player.x,player.y-20);
    ctx.lineTo(player.x-15,player.y+20);
    ctx.lineTo(player.x+15,player.y+20);
    ctx.fill();

    ctx.fillStyle="yellow";
    bullets.forEach(b=>ctx.fillRect(b.x-2,b.y,4,10));

    enemies.forEach(e=>{
        ctx.fillStyle="lime";
        ctx.beginPath();
        ctx.ellipse(e.x,e.y,20,10,0,0,Math.PI*2);
        ctx.fill();
    });

    if(boss){
        ctx.fillStyle="red";
        ctx.beginPath();
        ctx.ellipse(boss.x,boss.y,50,20,0,0,Math.PI*2);
        ctx.fill();

        ctx.fillStyle="white";
        ctx.fillText("BOSS:"+boss.hp,boss.x-30,boss.y-40);
    }

    ctx.fillStyle="orange";
    particles.forEach(p=>ctx.fillRect(p.x,p.y,3,3));

    ctx.fillStyle="white";
    ctx.fillText(`Score:${score} HP:${hp} Gun:${weaponLevel}`,10,20);

    if(state==="gameover"){
        ctx.font="40px Arial";
        ctx.fillText("GAME OVER",canvas.width/2-120,canvas.height/2);
        ctx.font="20px Arial";
        ctx.fillText("Press R",canvas.width/2-40,canvas.height/2+40);
    }
}

function loop(){
    update();
    draw();
    requestAnimationFrame(loop);
}

setInterval(spawnEnemy,800);
loop();
</script>

</body>
</html>
