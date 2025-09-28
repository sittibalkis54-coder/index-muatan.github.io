# index-muatan.github.io
<index-muatan.html>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simulasi Gaya Listrik AR</title>
    <!-- Menggunakan Tailwind CSS untuk styling UI -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            margin: 0;
            font-family: 'Inter', sans-serif;
            color: #e5e7eb; /* gray-200 */
        }
        #ar-button {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            padding: 12px 24px;
            border: none;
            border-radius: 8px;
            background-color: #4f46e5; /* indigo-600 */
            color: white;
            font-size: 16px;
            cursor: pointer;
            z-index: 100;
        }
        #ui-container {
            position: absolute;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            width: 90%;
            max-width: 400px;
            background-color: rgba(31, 41, 55, 0.8); /* gray-800 with opacity */
            backdrop-filter: blur(10px);
            border-radius: 12px;
            padding: 16px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            z-index: 50;
            display: none; /* Awalnya disembunyikan */
        }
        .charge-control {
            margin-bottom: 16px;
            background-color: rgba(55, 65, 81, 0.7); /* gray-700 with opacity */
            padding: 12px;
            border-radius: 8px;
        }
        .charge-control h3 {
            margin-top: 0;
            font-weight: 600;
            color: #d1d5db; /* gray-300 */
        }
        .info-display {
            background-color: rgba(17, 24, 39, 0.8); /* gray-900 with opacity */
            padding: 12px;
            border-radius: 8px;
            text-align: center;
        }
        .info-display p {
            margin: 4px 0;
            font-size: 14px;
        }
        .info-display span {
            font-weight: 700;
            color: #f9fafb; /* gray-50 */
        }
        input[type="range"] {
            -webkit-appearance: none;
            width: 100%;
            height: 8px;
            background: #4b5563; /* gray-600 */
            border-radius: 5px;
            outline: none;
            opacity: 0.7;
            transition: opacity .2s;
        }
        input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            appearance: none;
            width: 20px;
            height: 20px;
            background: #6366f1; /* indigo-500 */
            cursor: pointer;
            border-radius: 50%;
        }
        input[type="range"]::-moz-range-thumb {
            width: 20px;
            height: 20px;
            background: #6366f1; /* indigo-500 */
            cursor: pointer;
            border-radius: 50%;
        }
        #instructions {
            position: absolute;
            bottom: 80px;
            width: 100%;
            text-align: center;
            color: white;
            background-color: rgba(0,0,0,0.5);
            padding: 10px;
            z-index: 100;
            display: none; /* Initially hidden */
        }
    </style>
</head>
<body>
    <div id="ui-container">
        <!-- Kontrol untuk Muatan 1 -->
        <div class="charge-control">
            <h3>Muatan 1</h3>
            <div class="flex items-center justify-around my-2">
                <label><input type="radio" name="type1" value="1" checked> Positif (+)</label>
                <label><input type="radio" name="type1" value="-1"> Negatif (-)</label>
            </div>
            <input type="range" id="value1" min="1" max="10" value="5">
            <p class="text-center text-sm mt-1">Nilai: <span id="value1-label">5</span> µC</p>
        </div>

        <!-- Kontrol untuk Muatan 2 -->
        <div class="charge-control">
            <h3>Muatan 2</h3>
            <div class="flex items-center justify-around my-2">
                <label><input type="radio" name="type2" value="1" checked> Positif (+)</label>
                <label><input type="radio" name="type2" value="-1"> Negatif (-)</label>
            </div>
            <input type="range" id="value2" min="1" max="10" value="5">
            <p class="text-center text-sm mt-1">Nilai: <span id="value2-label">5</span> µC</p>
        </div>

        <!-- Tampilan Informasi -->
        <div class="info-display">
            <p>Jarak (r): <span id="distance-label">0.00</span> m</p>
            <p>Gaya Listrik (F): <span id="force-label">0.00</span> N</p>
        </div>
    </div>
    
    <div id="instructions">Gerakkan ponsel untuk mendeteksi permukaan datar, lalu ketuk layar untuk menempatkan muatan.</div>

    <script type="importmap">
        {
            "imports": {
                "three": "https://cdn.jsdelivr.net/npm/three@0.164.1/build/three.module.js",
                "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.164.1/examples/jsm/"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { ARButton } from 'three/addons/webxr/ARButton.js';

        let camera, scene, renderer;
        let controller;
        let reticle; // Objek untuk menandai posisi penempatan
        let hitTestSource = null;
        let hitTestSourceRequested = false;
        
        // Variabel untuk simulasi
        const charges = [];
        let forceArrow1, forceArrow2;
        const K_CONST = 8.99 * 1e9; // Konstanta Coulomb

        // Referensi UI
        const uiContainer = document.getElementById('ui-container');
        const instructions = document.getElementById('instructions');
        
        // Variabel untuk memindahkan objek
        let draggingObject = null;

        init();
        animate();

        function init() {
            const container = document.createElement('div');
            document.body.appendChild(container);

            scene = new THREE.Scene();
            camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.01, 20);

            const light = new THREE.HemisphereLight(0xffffff, 0xbbbbff, 3);
            light.position.set(0.5, 1, 0.25);
            scene.add(light);

            renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
            renderer.setPixelRatio(window.devicePixelRatio);
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.xr.enabled = true;
            container.appendChild(renderer.domElement);
            
            // Tombol AR
            const arButton = ARButton.createButton(renderer, {
                requiredFeatures: ['hit-test']
            });
            arButton.addEventListener('click', () => {
                instructions.style.display = 'block';
            });
            document.body.appendChild(arButton);
            

            // Setup Reticle (penanda)
            reticle = new THREE.Mesh(
                new THREE.RingGeometry(0.05, 0.07, 32).rotateX(-Math.PI / 2),
                new THREE.MeshBasicMaterial()
            );
            reticle.matrixAutoUpdate = false;
            reticle.visible = false;
            scene.add(reticle);

            // Setup Controller
            controller = renderer.xr.getController(0);
            controller.addEventListener('selectstart', onSelectStart);
            controller.addEventListener('selectend', onSelectEnd);
            scene.add(controller);

            // Setup Event Listener untuk UI
            setupUIListeners();

            window.addEventListener('resize', onWindowResize);
        }
        
        function onSelectStart(event) {
            if (reticle.visible && charges.length < 2) {
                placeCharge();
            } else if (charges.length === 2) {
                // Coba untuk memulai drag
                const controllerPosition = controller.getWorldPosition(new THREE.Vector3());
                let closestCharge = null;
                let minDistance = 0.1; // Jarak minimal untuk bisa di-drag

                charges.forEach(charge => {
                    const distance = charge.mesh.position.distanceTo(controllerPosition);
                    if (distance < minDistance) {
                        minDistance = distance;
                        closestCharge = charge;
                    }
                });

                if (closestCharge) {
                    draggingObject = closestCharge;
                    // Kaitkan objek ke controller
                    controller.attach(draggingObject.mesh);
                }
            }
        }

        function onSelectEnd(event) {
            if (draggingObject) {
                // Lepaskan objek dari controller ke scene
                scene.attach(draggingObject.mesh);
                draggingObject = null;
            }
        }


        function placeCharge() {
            const chargeIndex = charges.length;
            const type = document.querySelector(`input[name="type${chargeIndex + 1}"]:checked`).value;
            const value = document.getElementById(`value${chargeIndex + 1}`).value;
            
            const geometry = new THREE.SphereGeometry(0.04, 32, 32);
            const color = type == 1 ? 0xff0000 : 0x0000ff; // Merah untuk positif, Biru untuk negatif
            const material = new THREE.MeshStandardMaterial({ 
                color: color,
                roughness: 0.3,
                metalness: 0.2
            });
            
            const mesh = new THREE.Mesh(geometry, material);
            mesh.position.setFromMatrixPosition(reticle.matrix);
            scene.add(mesh);

            // Tambahkan outline untuk visibilitas
            const outlineGeo = new THREE.SphereGeometry(0.042, 32, 32);
            const outlineMat = new THREE.MeshBasicMaterial({ color: 0xffffff, side: THREE.BackSide });
            const outline = new THREE.Mesh(outlineGeo, outlineMat);
            mesh.add(outline);

            const charge = {
                mesh: mesh,
                type: parseInt(type),
                value: parseInt(value),
                index: chargeIndex + 1,
            };
            charges.push(charge);

            if (charges.length === 2) {
                uiContainer.style.display = 'block';
                instructions.style.display = 'none';
                reticle.visible = false;
                setupForceArrows();
            }
        }
        
        function setupForceArrows() {
            const dir = new THREE.Vector3(1, 0, 0);
            forceArrow1 = new THREE.ArrowHelper(dir, charges[0].mesh.position, 0, 0xff0000);
            forceArrow2 = new THREE.ArrowHelper(dir, charges[1].mesh.position, 0, 0xff0000);
            scene.add(forceArrow1);
            scene.add(forceArrow2);
        }

        function updatePhysicsAndArrows() {
            if (charges.length < 2) return;

            const charge1 = charges[0];
            const charge2 = charges[1];

            const pos1 = charge1.mesh.position;
            const pos2 = charge2.mesh.position;

            const distance = pos1.distanceTo(pos2);
            // Hindari pembagian dengan nol jika jarak terlalu kecil
            if (distance < 0.01) {
                document.getElementById('distance-label').textContent = 'Terlalu Dekat';
                document.getElementById('force-label').textContent = 'Tak Terhingga';
                forceArrow1.visible = false;
                forceArrow2.visible = false;
                return;
            }
            forceArrow1.visible = true;
            forceArrow2.visible = true;
            

            const q1 = charge1.type * charge1.value * 1e-6; // Konversi ke Coulomb
            const q2 = charge2.type * charge2.value * 1e-6;

            const forceMagnitude = K_CONST * Math.abs(q1 * q2) / (distance * distance);

            // Update UI
            document.getElementById('distance-label').textContent = `${distance.toFixed(2)} m`;
            document.getElementById('force-label').textContent = `${forceMagnitude.toExponential(2)} N`;

            // Update panah
            const isRepulsive = (q1 * q2) > 0; // Tolak-menolak jika tanda sama
            const arrowColor = isRepulsive ? 0xff0000 : 0x00ff00; // Merah untuk tolak, Hijau untuk tarik
            
            const direction1to2 = new THREE.Vector3().subVectors(pos2, pos1).normalize();
            const direction2to1 = new THREE.Vector3().subVectors(pos1, pos2).normalize();

            const arrowLength = Math.min(forceMagnitude * 1e-10, 0.5); // Skala panjang panah agar visual

            // Panah pada muatan 1 (gaya dari muatan 2)
            forceArrow1.position.copy(pos1);
            forceArrow1.setDirection(isRepulsive ? direction2to1 : direction1to2);
            forceArrow1.setLength(arrowLength, 0.1, 0.05);
            forceArrow1.setColor(arrowColor);
            
            // Panah pada muatan 2 (gaya dari muatan 1)
            forceArrow2.position.copy(pos2);
            forceArrow2.setDirection(isRepulsive ? direction1to2 : direction2to1);
            forceArrow2.setLength(arrowLength, 0.1, 0.05);
            forceArrow2.setColor(arrowColor);
        }

        function setupUIListeners() {
            ['1', '2'].forEach(index => {
                document.querySelectorAll(`input[name="type${index}"]`).forEach(radio => {
                    radio.addEventListener('change', (e) => {
                        const charge = charges[parseInt(index) - 1];
                        if (!charge) return;
                        charge.type = parseInt(e.target.value);
                        const newColor = charge.type === 1 ? 0xff0000 : 0x0000ff;
                        charge.mesh.material.color.setHex(newColor);
                    });
                });

                const slider = document.getElementById(`value${index}`);
                const label = document.getElementById(`value${index}-label`);
                slider.addEventListener('input', (e) => {
                    const value = e.target.value;
                    label.textContent = value;
                    const charge = charges[parseInt(index) - 1];
                    if (!charge) return;
                    charge.value = parseInt(value);
                });
            });
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        function animate() {
            renderer.setAnimationLoop(render);
        }

        function render(timestamp, frame) {
            if (frame) {
                const referenceSpace = renderer.xr.getReferenceSpace();
                const session = renderer.xr.getSession();

                if (hitTestSourceRequested === false) {
                    session.requestReferenceSpace('viewer').then(function (referenceSpace) {
                        session.requestHitTestSource({ space: referenceSpace }).then(function (source) {
                            hitTestSource = source;
                        });
                    });
                    session.addEventListener('end', function () {
                        hitTestSourceRequested = false;
                        hitTestSource = null;
                        // Reset aplikasi saat sesi AR berakhir
                        resetScene();
                    });
                    hitTestSourceRequested = true;
                }

                if (hitTestSource && charges.length < 2) {
                    const hitTestResults = frame.getHitTestResults(hitTestSource);
                    if (hitTestResults.length) {
                        const hit = hitTestResults[0];
                        reticle.visible = true;
                        reticle.matrix.fromArray(hit.getPose(referenceSpace).transform.matrix);
                    } else {
                        reticle.visible = false;
                    }
                }
            }
            
            updatePhysicsAndArrows();
            renderer.render(scene, camera);
        }

        function resetScene() {
            // Hapus muatan dan panah dari scene
            charges.forEach(charge => scene.remove(charge.mesh));
            if (forceArrow1) scene.remove(forceArrow1);
            if (forceArrow2) scene.remove(forceArrow2);
            
            // Kosongkan array dan variabel
            charges.length = 0;
            forceArrow1 = null;
            forceArrow2 = null;
            draggingObject = null;
            
            // Sembunyikan UI dan tampilkan instruksi lagi
            uiContainer.style.display = 'none';
            instructions.style.display = 'block';
        }

    </script>
</body>
</html>
