# expire-fighter
expire-fighter
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Pro Billiards - بلياردو للمحترفين</title>
    <style>
        body {
            margin: 0;
            background: radial-gradient(circle, #2c3e50, #1a252f);
            color: white;
            font-family: Tahoma, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            overflow: hidden;
            touch-action: none;
        }
        .scoreboard {
            display: flex;
            justify-content: space-between;
            width: 340px;
            background: rgba(0, 0, 0, 0.4);
            padding: 10px 15px;
            border-radius: 12px;
            margin-bottom: 10px;
            box-sizing: border-box;
            border: 1px solid rgba(255, 255, 255, 0.1);
        }
        .player {
            text-align: center;
            padding: 5px 12px;
            border-radius: 8px;
            transition: 0.3s;
            opacity: 0.5;
        }
        .player.active {
            opacity: 1;
            background: #27ae60;
            box-shadow: 0 0 10px rgba(39, 174, 96, 0.6);
            font-weight: bold;
        }
        canvas {
            background-color: #1e8449;
            border: 14px solid #4e342e;
            border-radius: 18px;
            box-shadow: inset 0 0 20px rgba(0,0,0,0.6), 0 12px 30px rgba(0,0,0,0.6);
        }
        .controls {
            margin-top: 12px;
            display: flex;
            gap: 10px;
            align-items: center;
        }
        button {
            background-color: #e67e22;
            color: white;
            border: none;
            padding: 10px 18px;
            border-radius: 8px;
            font-size: 14px;
            font-weight: bold;
            cursor: pointer;
            box-shadow: 0 4px 6px rgba(0,0,0,0.3);
        }
        button:active { transform: scale(0.96); }
        #statusMsg {
            font-size: 13px;
            color: #f1c40f;
            margin-bottom: 6px;
            height: 18px;
        }
    </style>
</head>
<body>

<div id="statusMsg">دور اللاعب 1 — اسحب الكرة البيضاء للضرب</div>

<div class="scoreboard">
    <div id="p1Box" class="player active">👤 اللاعب 1: <span id="p1Score">0</span></div>
    <div id="p2Box" class="player">👤 اللاعب 2: <span id="p2Score">0</span></div>
</div>

<canvas id="poolTable" width="320" height="480"></canvas>

<div class="controls">
    <button onclick="resetGame()">لعبة جديدة 🔄</button>
</div>

<script>
    const canvas = document.getElementById('poolTable');
    const ctx = canvas.getContext('2d');
    const statusMsg = document.getElementById('statusMsg');

    // مولد صوت بسيط للتصادم بدون ملفات خارجية
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    function playHitSound(volume = 0.3) {
        if (audioCtx.state === 'suspended') audioCtx.resume();
        const osc = audioCtx.createOscillator();
        const gain = audioCtx.createGain();
        osc.type = 'triangle';
        osc.frequency.setValueAtTime(220, audioCtx.currentTime);
        osc.frequency.exponentialRampToValueAtTime(50, audioCtx.currentTime + 0.06);
        gain.gain.setValueAtTime(Math.min(volume, 0.5), audioCtx.currentTime);
        gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.06);
        osc.connect(gain);
        gain.connect(audioCtx.destination);
        osc.start();
        osc.stop(audioCtx.currentTime + 0.06);
    }

    let currentPlayer = 1;
    let scores = { 1: 0, 2: 0 };
    let scoredThisTurn = false;
    let shotInProgress = false;

    const friction = 0.984;
    const ballRadius = 11;
    const pocketRadius = 18;

    const pockets = [
        { x: 14, y: 14 }, { x: canvas.width - 14, y: 14 },
        { x: 14, y: canvas.height / 2 }, { x: canvas.width - 14, y: canvas.height / 2 },
        { x: 14, y: canvas.height - 14 }, { x: canvas.width - 14, y: canvas.height - 14 }
    ];

    let balls = [];
    let cueBall = null;
    let isDragging = false;
    let dragStart = { x: 0, y: 0 };
    let dragCurrent = { x: 0, y: 0 };

    function resetGame() {
        currentPlayer = 1;
        scores = { 1: 0, 2: 0 };
        scoredThisTurn = false;
        shotInProgress = false;
        updateUI();

        cueBall = { x: canvas.width / 2, y: canvas.height - 95, vx: 0, vy: 0, color: '#ffffff', num: '', isCue: true };
        
        const ballData = [
            { color: '#f1c40f', num: '1' }, { color: '#2980b9', num: '2' },
            { color: '#e74c3c', num: '3' }, { color: '#8e44ad', num: '4' },
            { color: '#111111', num: '8' }, { color: '#e67e22', num: '5' },
            { color: '#16a085', num: '6' }, { color: '#c0392b', num: '7' },
            { color: '#d35400', num: '9' }, { color: '#2c3e50', num: '10' }
        ];

        balls = [cueBall];
        let startX = canvas.width / 2;
        let startY = 145;
        let idx = 0;

        for (let row = 0; row < 4; row++) {
            for (let col = 0; col <= row; col++) {
                if (idx < ballData.length) {
                    balls.push({
                        x: startX + (col - row / 2) * (ballRadius * 2.2),
                        y: startY - row * (ballRadius * 2),
                        vx: 0,
                        vy: 0,
                        color: ballData[idx].color,
                        num: ballData[idx].num,
                        isCue: false
                    });
                    idx++;
                }
            }
        }
    }

    function updateUI() {
        document.getElementById('p1Score').innerText = scores[1];
        document.getElementById('p2Score').innerText = scores[2];
        document.getElementById('p1Box').classList.toggle('active', currentPlayer === 1);
        document.getElementById('p2Box').classList.toggle('active', currentPlayer === 2);
        statusMsg.innerText = `دور اللاعب ${currentPlayer} — اسحب الكرة البيضاء للضرب`;
    }

    function getPos(e) {
        const rect = canvas.getBoundingClientRect();
        const clientX = e.touches ? e.touches[0].clientX : e.clientX;
        const clientY = e.touches ? e.touches[0].clientY : e.clientY;
        return { x: clientX - rect.left, y: clientY - rect.top };
    }

    function isMoving() {
        return balls.some(b => Math.abs(b.vx) > 0.05 || Math.abs(b.vy) > 0.05);
    }

    canvas.addEventListener('mousedown', startDrag);
    canvas.addEventListener('touchstart', startDrag);

    function startDrag(e) {
        if (isMoving() || balls.length === 1) return;
        const pos = getPos(e);
        if (Math.hypot(pos.x - cueBall.x, pos.y - cueBall.y) < 50) {
            isDragging = true;
            dragStart = { x: cueBall.x, y: cueBall.y };
            dragCurrent = pos;
        }
    }

    window.addEventListener('mousemove', e => { if (isDragging) dragCurrent = getPos(e); });
    window.addEventListener('touchmove', e => { if (isDragging) dragCurrent = getPos(e); });

    window.addEventListener('mouseup', endDrag);
    window.addEventListener('touchend', endDrag);

    function endDrag() {
        if (!isDragging) return;
        isDragging = false;
        const dx = dragStart.x - dragCurrent.x;
        const dy = dragStart.y - dragCurrent.y;
        const power = Math.min(Math.hypot(dx, dy) * 0.13, 17);
        const angle = Math.atan2(dy, dx);

        if (power > 1.2) {
            cueBall.vx = Math.cos(angle) * power;
            cueBall.vy = Math.sin(angle) * power;
            scoredThisTurn = false;
            shotInProgress = true;
            playHitSound(power / 20);
        }
    }

    function update() {
        for (let i = balls.length - 1; i >= 0; i--) {
            let b = balls[i];
            b.x += b.vx;
            b.y += b.vy;
            b.vx *= friction;
            b.vy *= friction;

            if (Math.abs(b.vx) < 0.05) b.vx = 0;
            if (Math.abs(b.vy) < 0.05) b.vy = 0;

            if (b.x < ballRadius) { b.x = ballRadius; b.vx *= -0.8; playHitSound(Math.abs(b.vx)/15); }
            if (b.x > canvas.width - ballRadius) { b.x = canvas.width - ballRadius; b.vx *= -0.8; playHitSound(Math.abs(b.vx)/15); }
            if (b.y < ballRadius) { b.y = ballRadius; b.vy *= -0.8; playHitSound(Math.abs(b.vy)/15); }
            if (b.y > canvas.height - ballRadius) { b.y = canvas.height - ballRadius; b.vy *= -0.8; playHitSound(Math.abs(b.vy)/15); }

            for (let p of pockets) {
                if (Math.hypot(b.x - p.x, b.y - p.y) < pocketRadius) {
                    if (b.isCue) {
                        b.x = canvas.width / 2;
                        b.y = canvas.height - 95;
                        b.vx = 0; b.vy = 0;
                        scoredThisTurn = false;
                    } else {
                        balls.splice(i, 1);
                        scores[currentPlayer] += 10;
                        scoredThisTurn = true;
                        playHitSound(0.4);
                    }
                    break;
                }
            }
        }

        // تصادم الكرات
        for (let i = 0; i < balls.length; i++) {
            for (let j = i + 1; j < balls.length; j++) {
                let b1 = balls[i];
                let b2 = balls[j];
                let dx = b2.x - b1.x;
                let dy = b2.y - b1.y;
                let dist = Math.hypot(dx, dy);

                if (dist < ballRadius * 2) {
                    let angle = Math.atan2(dy, dx);
                    let overlap = (ballRadius * 2 - dist) / 2;
                    b1.x -= Math.cos(angle) * overlap;
                    b1.y -= Math.sin(angle) * overlap;
                    b2.x += Math.cos(angle) * overlap;
                    b2.y += Math.sin(angle) * overlap;

                    let nx = dx / dist;
                    let ny = dy / dist;
                    let p = (b1.vx * nx + b1.vy * ny - b2.vx * nx - b2.vy * ny);

                    if (p > 0) {
                        b1.vx -= p * nx;
                        b1.vy -= p * ny;
                        b2.vx += p * nx;
                        b2.vy += p * ny;
                        playHitSound(p / 15);
                    }
                }
            }
        }

        // تبديل الدور بعد توقف جميع الكرات
        if (shotInProgress && !isMoving()) {
            shotInProgress = false;
            if (!scoredThisTurn && balls.length > 1) {
                currentPlayer = currentPlayer === 1 ? 2 : 1;
            }
            updateUI();
        }
    }

    function draw() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        // خط البداية على الطاولة
        ctx.strokeStyle = 'rgba(255,255,255,0.15)';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.moveTo(0, canvas.height - 95);
        ctx.lineTo(canvas.width, canvas.height - 95);
        ctx.stroke();

        // الحفر
        ctx.fillStyle = '#0d1117';
        for (let p of pockets) {
            ctx.beginPath();
            ctx.arc(p.x, p.y, pocketRadius, 0, Math.PI * 2);
            ctx.fill();
        }

        // رسم خط التصويب وعصا البلياردو
        if (isDragging) {
            const dx = dragStart.x - dragCurrent.x;
            const dy = dragStart.y - dragCurrent.y;
            const angle = Math.atan2(dy, dx);
            const pullBack = Math.min(Math.hypot(dx, dy), 110);

            // خط التصويب
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.65)';
            ctx.lineWidth = 2;
            ctx.setLineDash([6, 6]);
            ctx.beginPath();
            ctx.moveTo(cueBall.x, cueBall.y);
            ctx.lineTo(cueBall.x + Math.cos(angle) * 180, cueBall.y + Math.sin(angle) * 180);
            ctx.stroke();
            ctx.setLineDash([]);

            // عصا البلياردو
            ctx.save();
            ctx.translate(cueBall.x, cueBall.y);
            ctx.rotate(angle + Math.PI);
            ctx.fillStyle = '#d7ccc8';
            ctx.fillRect(16 + pullBack * 0.4, -2.5, 110, 5);
            ctx.fillStyle = '#3e2723';
            ctx.fillRect(50 + pullBack * 0.4, -3, 76, 6);
            ctx.restore();
        }

        // رسم الكرات بأرقامها
        for (let b of balls) {
            ctx.beginPath();
            ctx.arc(b.x, b.y, ballRadius, 0, Math.PI * 2);
            ctx.fillStyle = b.color;
            ctx.fill();

            if (!b.isCue) {
                ctx.beginPath();
                ctx.arc(b.x, b.y, 5.5, 0, Math.PI * 2);
                ctx.fillStyle = 'white';
                ctx.fill();

                ctx.fillStyle = 'black';
                ctx.font = 'bold 8px Arial';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.fillText(b.num, b.x, b.y + 0.5);
            }
        }

        // شاشة الفوز
        if (balls.length === 1) {
            ctx.fillStyle = 'rgba(0, 0, 0, 0.8)';
            ctx.fillRect(15, canvas.height / 2 - 50, canvas.width - 30, 100);
            ctx.fillStyle = '#f1c40f';
            ctx.font = 'bold 20px Tahoma';
            ctx.textAlign = 'center';
            let winner = scores[1] > scores[2] ? '🏆 فاز اللاعب 1!' : (scores[2] > scores[1] ? '🏆 فاز اللاعب 2!' : '🤝 تعادل!');
            ctx.fillText(winner, canvas.width / 2, canvas.height / 2 - 5);
            ctx.fillStyle = 'white';
            ctx.font = '14px Tahoma';
            ctx.fillText(`النتيجة: ${scores[1]} - ${scores[2]}`, canvas.width / 2, canvas.height / 2 + 25);
        }
    }

    function loop() {
        update();
        draw();
        requestAnimationFrame(loop);
    }

    resetGame();
    loop();
</script>
</body>
</html>
