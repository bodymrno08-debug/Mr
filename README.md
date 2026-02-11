<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hand-Controlled 3D Particle System</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #050505;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            user-select: none;
        }

        /* The 3D Canvas */
        canvas {
            display: block;
            position: absolute;
            top: 0;
            left: 0;
            z-index: 1;
        }

        /* UI Overlay */
        #ui-layer {
            position: absolute;
            top: 20px;
            left: 20px;
            z-index: 10;
            color: rgba(255, 255, 255, 0.9);
            pointer-events: none;
        }

        h1 {
            margin: 0 0 5px 0;
            font-size: 1.8rem;
            text-transform: uppercase;
            letter-spacing: 2px;
            background: linear-gradient(90deg, #00ff88, #00d4ff);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }

        .status-box {
            font-size: 0.9rem;
            margin-bottom: 15px;
            padding: 5px 10px;
            background: rgba(0, 0, 0, 0.5);
            border-left: 3px solid #00ff88;
            border-radius: 4px;
            display: inline-block;
        }

        .instructions {
            font-size: 0.85rem;
            line-height: 1.6;
            background: rgba(0,0,0,0.3);
            padding: 10px;
            border-radius: 8px;
            backdrop-filter: blur(4px);
            max-width: 250px;
        }

        .key { color: #00ff88; font-weight: bold; }

        /* Button Controls at Bottom */
        #controls {
            position: absolute;
            bottom: 30px;
            width: 100%;
            display: flex;
            justify-content: center;
            gap: 10px;
            z-index: 10;
            pointer-events: auto;
        }

        button {
            background: rgba(255, 255, 255, 0.1);
            border: 1px solid rgba(255, 255, 255, 0.2);
            color: white;
            padding: 10px 20px;
            border-radius: 30px;
            cursor: pointer;
            backdrop-filter: blur(10px);
            transition: all 0.3s ease;
            font-weight: 600;
            text-transform: uppercase;
            font-size: 0.8rem;
        }

        button:hover {
            background: rgba(255, 255, 255, 0.25);
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(0, 255, 136, 0.2);
        }

        button.active {
            border-color: #00ff88;
            background: rgba(0, 255, 136, 0.1);
            color: #00ff88;
            box-shadow: 0 0 10px rgba(0, 255, 136, 0.2);
        }

        /* Loading Spinner */
        #loader {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: #00ff88;
            font-size: 1.2rem;
            text-align: center;
            z-index: 20;
            background: rgba(0,0,0,0.8);
            padding: 20px;
            border-radius: 10px;
        }

        /* Hidden video element for MediaPipe input */
        #input_video {
            display: none;
        }
    </style>
</head>
<body>

    <div id="ui-layer">
        <h1>Particle Morph</h1>
        <div class="status-box" id="status">Initializing AI Model...</div>
        
        <div class="instructions">
            <div><span class="key">Move Hand:</span> Rotate & Change Color</div>
            <div><span class="key">Pinch (Index+Thumb):</span> Implode</div>
            <div><span class="key">Open Hand:</span> Explode/Expand</div>
            <div><span class="key">Two Hands:</span> Chaos Mode</div>
        </div>
    </div>

    <div id="loader">Starting Camera & AI...<br><small>(Please allow camera access)</small></div>

    <div id="controls">
        <button onclick="setTargetShape('sphere')" class="active" id="btn-sphere">Sphere</button>
        <button onclick="setTargetShape('heart')" id="btn-heart">Heart</button>
        <button onclick="setTargetShape('flower')" id="btn-flower">Flower</button>
        <button onclick="setTargetShape('saturn')" id="btn-saturn">Saturn</button>
        <button onclick="setTargetShape('helix')" id="btn-helix">Helix</button>
    </div>

    <video id="input_video"></video>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>

    <script>
        // --- 1. CONFIGURATION & STATE ---
        const CONFIG = {
            count: 15000,          // Number of particles
            size: 0.2,             // Particle size
            lerpSpeed: 0.08,       // How fast particles morph shapes
            colorSpeed: 0.05       // How fast colors change
        };

        const state = {
            handPresent: false,
            handX: 0,              // Normalized -1 to 1
            handY: 0,              // Normalized -1 to 1
            pinchStrength: 0,      // 0 (open) to 1 (closed)
            twoHands: false
        };

        // Three.js Variables
        let scene, camera, renderer, particles;
        let positions, targetPositions, currentColors;
        let clock = new THREE.Clock();
        let time = 0;

        // --- 2. THREE.JS SETUP ---
        function initThreeJS() {
            scene = new THREE.Scene();
            scene.fog = new THREE.FogExp2(0x050505, 0.02);

            camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.z = 35;

            renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.body.appendChild(renderer.domElement);

            createParticles();
            setTargetShape('sphere'); // Initial shape

            window.addEventListener('resize', () => {
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
            });

            animate();
        }

        // --- 3. PARTICLE SYSTEM ---
        function createParticles() {
            const geometry = new THREE.BufferGeometry();
            positions = new Float32Array(CONFIG.count * 3);
            targetPositions = new Float32Array(CONFIG.count * 3);
            currentColors = new Float32Array(CONFIG.count * 3);
            
            for(let i=0; i<CONFIG.count * 3; i++) {
                positions[i] = (Math.random() - 0.5) * 100;
                currentColors[i] = 1.0; 
            }

            geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
            geometry.setAttribute('color', new THREE.BufferAttribute(currentColors, 3));

            const textureLoader = new THREE.TextureLoader();
            const particleTexture = textureLoader.load('https://threejs.org/examples/textures/sprites/spark1.png');

            const material = new THREE.PointsMaterial({
                size: CONFIG.size,
                map: particleTexture,
                vertexColors: true,
                blending: THREE.AdditiveBlending,
                depthWrite: false,
                transparent: true,
                opacity: 0.8
            });

            particles = new THREE.Points(geometry, material);
            scene.add(particles);
        }

        // --- 4. SHAPE GENERATORS ---
        window.setTargetShape = function(type) {
            document.querySelectorAll('#controls button').forEach(b => b.classList.remove('active'));
            const btn = document.getElementById(`btn-${type}`);
            if(btn) btn.classList.add('active');

            const scale = 15;
            const count = CONFIG.count;

            for (let i = 0; i < count; i++) {
                const idx = i * 3;
                let x, y, z;

                if (type === 'sphere') {
                    const phi = Math.acos(-1 + (2 * i) / count);
                    const theta = Math.sqrt(count * Math.PI) * phi;
                    x = scale * Math.cos(theta) * Math.sin(phi);
                    y = scale * Math.sin(theta) * Math.sin(phi);
                    z = scale * Math.cos(phi);
                } 
                else if (type === 'heart') {
                    const phi = Math.random() * Math.PI * 2;
                    const theta = Math.random() * Math.PI;
                    x = 16 * Math.pow(Math.sin(theta), 3) * Math.cos(phi);
                    y = 13 * Math.cos(theta) - 5 * Math.cos(2*theta) - 2 * Math.cos(3*theta) - Math.cos(4*theta);
                    z = x * Math.sin(phi); 
                    x *= 0.6; y *= 0.6; z *= 0.6;
                }
                else if (type === 'flower') {
                    const u = Math.random() * Math.PI * 2;
                    const v = Math.random() * Math.PI;
                    const r = scale * Math.sin(3 * u) * Math.sin(v); 
                    x = r * Math.cos(u);
                    y = r * Math.sin(u);
                    z = (scale * 0.3) * Math.cos(v);
                }
                else if (type === 'saturn') {
                    if (i < count * 0.6) {
                        const phi = Math.acos(-1 + (2 * i) / (count * 0.6));
                        const theta = Math.sqrt((count * 0.6) * Math.PI) * phi;
                        const r = scale * 0.5;
                        x = r * Math.cos(theta) * Math.sin(phi);
                        y = r * Math.sin(theta) * Math.sin(phi);
                        z = r * Math.cos(phi);
                    } else {
                        const angle = Math.random() * Math.PI * 2;
                        const dist = (scale * 0.8) + Math.random() * scale;
                        x = dist * Math.cos(angle);
                        z = dist * Math.sin(angle);
                        y = (Math.random()-0.5) * 1;
                        
                        const tilt = 0.5;
                        const _y = y * Math.cos(tilt) - z * Math.sin(tilt);
                        const _z = y * Math.sin(tilt) + z * Math.cos(tilt);
                        y = _y; z = _z;
                    }
                }
                else if (type === 'helix') {
                    const t = i / count * 20 * Math.PI;
                    x = (scale * 0.6) * Math.cos(t);
                    z = (scale * 0.6) * Math.sin(t);
                    y = (i / count - 0.5) * scale * 2.5;
                }

                targetPositions[idx] = x;
                targetPositions[idx+1] = y;
                targetPositions[idx+2] = z;
            }
        };

        // --- 5. ANIMATION LOOP ---
        function animate() {
            requestAnimationFrame(animate);
            
            time += 0.01;
            const positionsArr = particles.geometry.attributes.position.array;
            const colorsArr = particles.geometry.attributes.color.array;

            if (state.handPresent) {
                particles.rotation.y += state.handX * 0.03;
                particles.rotation.x += -state.handY * 0.03;
            } else {
                particles.rotation.y += 0.002;
            }

            for(let i=0; i < CONFIG.count; i++) {
                const idx = i * 3;
                
                let tx = targetPositions[idx];
                let ty = targetPositions[idx+1];
                let tz = targetPositions[idx+2];

                if (state.handPresent) {
                    if (state.twoHands) {
                        tx += Math.sin(time * 10 + i) * 5;
                        ty += Math.cos(time * 10 + i) * 5;
                    } 
                    else if (state.pinchStrength > 0.8) {
                        tx *= 0.1; ty *= 0.1; tz *= 0.1;
                    } 
                    else if (state.pinchStrength < 0.2) {
                        tx *= 1.3; ty *= 1.3; tz *= 1.3;
                    }
                }

                const noise = Math.sin(time * 2 + i * 0.1) * 0.05;
                positionsArr[idx] += (tx - positionsArr[idx]) * CONFIG.lerpSpeed + noise;
                positionsArr[idx+1] += (ty - positionsArr[idx+1]) * CONFIG.lerpSpeed + noise;
                positionsArr[idx+2] += (tz - positionsArr[idx+2]) * CONFIG.lerpSpeed + noise;

                // Color Animation
                if (state.handPresent) {
                    const hue = (state.handX + 1) * 0.5 + (positionsArr[idx+1] * 0.02);
                    const col = new THREE.Color().setHSL(hue % 1, 0.8, 0.6);
                    colorsArr[idx] = THREE.MathUtils.lerp(colorsArr[idx], col.r, CONFIG.colorSpeed);
                    colorsArr[idx+1] = THREE.MathUtils.lerp(colorsArr[idx+1], col.g, CONFIG.colorSpeed);
                    colorsArr[idx+2] = THREE.MathUtils.lerp(colorsArr[idx+2], col.b, CONFIG.colorSpeed);
                } else {
                    const hue = 0.5 + (positionsArr[idx+1] * 0.02);
                    const col = new THREE.Color().setHSL(hue, 0.7, 0.5);
                    colorsArr[idx] = THREE.MathUtils.lerp(colorsArr[idx], col.r, 0.02);
                    colorsArr[idx+1] = THREE.MathUtils.lerp(colorsArr[idx+1], col.g, 0.02);
                    colorsArr[idx+2] = THREE.MathUtils.lerp(colorsArr[idx+2], col.b, 0.02);
                }
            }

            particles.geometry.attributes.position.needsUpdate = true;
            particles.geometry.attributes.color.needsUpdate = true;
            renderer.render(scene, camera);
        }

        // --- 6. MEDIAPIPE AI SETUP ---
        function initMediaPipe() {
            const videoElement = document.getElementById('input_video');
            const statusEl = document.getElementById('status');
            const loader = document.getElementById('loader');

            const hands = new Hands({locateFile: (file) => {
                return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
            }});

            hands.setOptions({
                maxNumHands: 2,
                modelComplexity: 1,
                minDetectionConfidence: 0.5,
                minTrackingConfidence: 0.5
            });

            hands.onResults((results) => {
                if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                    state.handPresent = true;
                    state.twoHands = results.multiHandLandmarks.length > 1;

                    statusEl.innerText = "Tracking Active";
                    statusEl.style.borderColor = "#00ff88";

                    const lm = results.multiHandLandmarks[0];
                    
                    const cx = (lm[0].x + lm[5].x + lm[17].x) / 3;
                    const cy = (lm[0].y + lm[5].y + lm[17].y) / 3;

                    state.handX = (1 - cx) * 2 - 1;
                    state.handY = -(cy * 2 - 1);

                    const thumb = lm[4];
                    const index = lm[8];
                    const dist = Math.hypot(thumb.x - index.x, thumb.y - index.y);
                    
                    const rawPinch = (0.15 - dist) * 10;
                    state.pinchStrength = Math.max(0, Math.min(1, rawPinch));

                } else {
                    state.handPresent = false;
                    state.twoHands = false;
                    statusEl.innerText = "No Hand Detected";
                    statusEl.style.borderColor = "#ff4444";
                }
            });

            const camera = new Camera(videoElement, {
                onFrame: async () => {
                    await hands.send({image: videoElement});
                },
                width: 640,
                height: 480
            });
            
            camera.start()
                .then(() => {
                    console.log("Camera started");
                    loader.style.display = 'none';
                })
                .catch(err => {
                    console.error("Camera failed", err);
                    loader.innerHTML = "Error: Camera Access Denied.<br>Please allow camera access and refresh.";
                    loader.style.color = "red";
                });
        }

        // --- 7. START APPLICATION ---
        initThreeJS();
        initMediaPipe();

    </script>
</body>
</html>
