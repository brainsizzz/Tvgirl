<!DOCTYPE html>
<html lang="az">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D ASCII Orbiting Planets</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #030408;
            overflow: hidden;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        canvas {
            display: block;
        }
    </style>
</head>
<body>

<canvas id="asciiCanvas"></canvas>

<script>
const canvas = document.getElementById('asciiCanvas');
const ctx = canvas.getContext('2d');

// Ekran ölçüləri və simvol toru (grid)
let cols, rows;
const charWidth = 8;
const charHeight = 14;

// İşıq intensivliyinə görə ASCII simvolları (tünddən parlağa)
const chars = " .:-=+*#%@";

// Arxa plan ulduzları
let stars = [];

function resize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    cols = Math.floor(canvas.width / charWidth);
    rows = Math.floor(canvas.height / charHeight);
    initStars();
}

function initStars() {
    stars = [];
    const starCount = Math.floor((cols * rows) * 0.015);
    for (let i = 0; i < starCount; i++) {
        stars.push({
            x: Math.floor(Math.random() * cols),
            y: Math.floor(Math.random() * rows),
            phase: Math.random() * Math.PI * 2,
            speed: 0.03 + Math.random() * 0.05,
            isCross: Math.random() > 0.7 // Bəzi ulduzların + şəklində parlaması üçün
        });
    }
}

window.addEventListener('resize', resize);
resize();

// İşıq mənbəyinin istiqaməti (normallaşdırılmış vektor)
const lightDir = { x: -0.5, y: -0.4, z: -0.75 };
const lightLen = Math.hypot(lightDir.x, lightDir.y, lightDir.z);
lightDir.x /= lightLen;
lightDir.y /= lightLen;
lightDir.z /= lightLen;

let time = 0;

function render() {
    ctx.fillStyle = '#030408';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.font = 'bold 13px monospace';
    ctx.textBaseline = 'top';

    // Z-buffer və simvol buferi
    const zBuffer = new Float32Array(cols * rows).fill(Infinity);
    const charBuffer = new Array(cols * rows).fill(' ');
    const colorBuffer = new Array(cols * rows).fill('#ffffff');

    // 1. Ulduzların hesablanması
    for (let s of stars) {
        s.phase += s.speed;
        const brightness = (Math.sin(s.phase) + 1) / 2;
        if (brightness > 0.2) {
            const idx = s.y * cols + s.x;
            const alpha = Math.floor(brightness * 220);
            const color = `rgb(${alpha}, ${alpha}, ${Math.min(255, alpha + 35)})`;
            
            if (s.isCross && brightness > 0.75) {
                charBuffer[idx] = '+';
                colorBuffer[idx] = color;
                // Parlaq ulduzun ətrafındakı şüalar
                if (s.x > 0) { charBuffer[idx - 1] = '-'; colorBuffer[idx - 1] = color; }
                if (s.x < cols - 1) { charBuffer[idx + 1] = '-'; colorBuffer[idx + 1] = color; }
                if (s.y > 0) { charBuffer[idx - cols] = '|'; colorBuffer[idx - cols] = color; }
                if (s.y < rows - 1) { charBuffer[idx + cols] = '|'; colorBuffer[idx + cols] = color; }
            } else {
                charBuffer[idx] = brightness > 0.6 ? '*' : '.';
                colorBuffer[idx] = color;
            }
        }
    }

    // 2. İki planetin orbit koordinatları
    const orbitRadiusX = Math.min(cols * 0.25, 32);
    const orbitRadiusZ = 18;

    const spheres = [
        {
            // Böyük, isti/narıncı tonlu kürə
            cx: Math.cos(time) * orbitRadiusX,
            cy: Math.sin(time * 0.5) * 2,
            cz: 45 + Math.sin(time) * orbitRadiusZ,
            r: 13.5,
            type: 'warm'
        },
        {
            // Kiçik, mavi tonlu kürə
            cx: Math.cos(time + Math.PI) * orbitRadiusX,
            cy: -Math.sin(time * 0.5) * 2,
            cz: 45 + Math.sin(time + Math.PI) * orbitRadiusZ,
            r: 10.5,
            type: 'blue'
        }
    ];

    const fov = 45;
    const aspectCorrection = charHeight / charWidth; // Simvolların hündürlük/en nisbətini tarazlayır

    // 3. Kürələrin 3D səthinin hesablanması
    for (let sp of spheres) {
        // Səthi sıx nöqtələrlə skan edirik
        for (let theta = 0; theta < Math.PI; theta += 0.045) {
            for (let phi = 0; phi < Math.PI * 2; phi += 0.045) {
                // Kürə üzərində lokal normal vektor
                const nx = Math.sin(theta) * Math.cos(phi);
                const ny = Math.cos(theta);
                const nz = Math.sin(theta) * Math.sin(phi);

                // Fəzadakı 3D koordinat
                const wx = sp.cx + sp.r * nx;
                const wy = sp.cy + sp.r * ny;
                const wz = sp.cz + sp.r * nz;

                if (wz <= 1) continue;

                // 2D ekran koordinatlarına proyeksiya
                const scale = fov / wz;
                const sx = Math.floor(cols / 2 + wx * scale * aspectCorrection);
                const sy = Math.floor(rows / 2 + wy * scale);

                if (sx >= 0 && sx < cols && sy >= 0 && sy < rows) {
                    const idx = sy * cols + sx;
                    // Z-buffer yoxlanışı (öndə olan kürə arxadakını örtür)
                    if (wz < zBuffer[idx]) {
                        zBuffer[idx] = wz;

                        // Səth teksturası (sətirvari dalğalar və fırlanma effekti)
                        const pattern = Math.sin(ny * 18 + time * 2) * 0.15 + Math.cos(phi * 3 + time * 3) * 0.1;
                        
                        // İşıqlanma (Diffuse lighting)
                        let diff = -(nx * lightDir.x + ny * lightDir.y + nz * lightDir.z) + pattern;
                        diff = Math.max(0.08, Math.min(1, diff));

                        const charIdx = Math.floor(diff * (chars.length - 1));
                        charBuffer[idx] = chars[charIdx];

                        // Rəngləndirmə
                        if (sp.type === 'blue') {
                            const r = Math.floor(20 + diff * 60);
                            const g = Math.floor(90 + diff * 130);
                            const b = Math.floor(180 + diff * 75);
                            colorBuffer[idx] = `rgb(${r}, ${g}, ${b})`;
                        } else {
                            // Narıncı, qəhvəyi və mavi zolaqlı planet rəngləri (Yupiter/Yer qarışığı kimi)
                            const band = Math.sin(ny * 10 + Math.sin(phi * 2 + time));
                            if (band > 0.35) {
                                // Mavi zolaqlar
                                const r = Math.floor(40 + diff * 70);
                                const g = Math.floor(100 + diff * 110);
                                const b = Math.floor(170 + diff * 85);
                                colorBuffer[idx] = `rgb(${r}, ${g}, ${b})`;
                            } else {
                                // İsti torpaq/narıncı tonlar
                                const r = Math.floor(140 + diff * 115);
                                const g = Math.floor(85 + diff * 95);
                                const b = Math.floor(55 + diff * 65);
                                colorBuffer[idx] = `rgb(${r}, ${g}, ${b})`;
                            }
                        }
                    }
                }
            }
        }
    }

    // 4. Bütün buferi ekrana çəkmək
    for (let y = 0; y < rows; y++) {
        for (let x = 0; x < cols; x++) {
            const idx = y * cols + x;
            const ch = charBuffer[idx];
            if (ch !== ' ') {
                ctx.fillStyle = colorBuffer[idx];
                ctx.fillText(ch, x * charWidth, y * charHeight);
            }
        }
    }

    time += 0.025; // Fırlanma sürəti
    requestAnimationFrame(render);
}

render();
</script>
</body>
</html>
