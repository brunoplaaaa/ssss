# ssss
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Project</title>
    
    <style>
body {
    font-family: Arial, sans-serif;
    margin: 0;
    padding: 20px;
    background: #f5f5f5;
}

h1 {
    color: #333;
}
    </style>
</head>
<body>
https://www.web-leb.com/en/editor/3811<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>WebXR Hit Test - Reacción Química AR</title>
    
    <!-- Three.js y extensiones -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/renderers/CSS2DRenderer.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/utils/BufferGeometryUtils.js"></script>
    
    <style>
        body {
            margin: 0;
            overflow: hidden;
            font-family: 'Arial', sans-serif;
            background: black;
            touch-action: none; /* Evita scroll con gestos */
        }
        #info-panel {
            position: absolute;
            bottom: 30px;
            left: 30px;
            width: 280px;
            background: rgba(10, 20, 30, 0.9);
            backdrop-filter: blur(5px);
            border-left: 6px solid #00ccff;
            border-radius: 12px;
            padding: 18px 22px;
            color: white;
            font-size: 16px;
            box-shadow: 0 8px 20px rgba(0,0,0,0.5);
            border: 1px solid rgba(255,255,255,0.2);
            transition: opacity 0.3s ease;
            pointer-events: none;
            z-index: 100;
        }
        #info-panel h3 {
            margin: 0 0 10px 0;
            color: #88ddff;
            font-weight: 600;
            font-size: 20px;
            border-bottom: 1px solid #4488aa;
            padding-bottom: 6px;
        }
        #info-panel p {
            margin: 8px 0;
            line-height: 1.5;
        }
        .highlight {
            color: #ffaa66;
            font-weight: bold;
        }
        .hidden-panel {
            opacity: 0;
            visibility: hidden;
        }
        .visible-panel {
            opacity: 1;
            visibility: visible;
        }
        #status {
            position: absolute;
            top: 30px;
            right: 30px;
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 12px 18px;
            border-radius: 30px;
            font-size: 16px;
            border: 1px solid #00ccff;
            backdrop-filter: blur(2px);
            pointer-events: none;
            z-index: 200;
        }
        #instructions {
            position: absolute;
            top: 30px;
            left: 30px;
            background: rgba(0,0,0,0.6);
            color: #ccc;
            padding: 12px 18px;
            border-radius: 8px;
            font-size: 14px;
            pointer-events: none;
            z-index: 200;
        }
        #ar-button {
            position: absolute;
            bottom: 30px;
            right: 30px;
            background: #00ccff;
            color: black;
            padding: 16px 24px;
            border-radius: 40px;
            font-size: 18px;
            font-weight: bold;
            border: none;
            box-shadow: 0 4px 10px rgba(0,0,0,0.3);
            pointer-events: auto;
            z-index: 300;
            cursor: pointer;
        }
        canvas {
            display: block;
        }
    </style>
</head>
<body>
    <div id="info-panel" class="hidden-panel">
        <h3 id="particle-title">Oxígeno (O)</h3>
        <p><span class="highlight">Número atómico:</span> <span id="atomic-number">8</span></p>
        <p><span class="highlight">Masa atómica:</span> <span id="atomic-mass">15.999 u</span></p>
        <p><span class="highlight">Electronegatividad:</span> <span id="electronegativity">3.44</span></p>
        <p><span class="highlight">Descubrimiento:</span> <span id="discovery">1774 (Priestley/Scheele)</span></p>
        <p style="margin-top:12px; font-style:italic; color:#aaa;" id="extra-info">Esencial para la respiración celular.</p>
    </div>

    <div id="status">🌍 Buscando superficies...</div>
    <div id="instructions">
        👆 Toca en una superficie para colocar Oxígeno (rojo)<br>
        👆 Toca en otra superficie para colocar Hidrógeno (blanco)<br>
        🖐️ Arrastra con 1 dedo para mover<br>
        ✌️ Dos dedos para escalar<br>
        👆👆 Doble tap para info
    </div>
    <button id="ar-button">Iniciar AR</button>

    <script>
        // --- VARIABLES GLOBALES ---
        const infoPanel = document.getElementById('info-panel');
        const statusDiv = document.getElementById('status');
        const arButton = document.getElementById('ar-button');

        // --- ESTADO DE LA APLICACIÓN ---
        let xrSession = null;
        let xrReferenceSpace = null;
        let xrHitTestSource = null;

        // --- MODELOS 3D ---
        let oxygenModel = null;
        let hydrogenModel = null;
        let waterModel = null;

        // --- INTERACCIÓN ---
        let selectedObject = null; // Objeto seleccionado para arrastrar
        let isDragging = false;
        let initialDistance = 0;
        let initialScale = 1;
        let lastTap = 0;

        // --- Three.js setup ---
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.01, 20);
        camera.position.set(0, 1.6, 0); // Altura de cámara aproximada

        const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.xr.enabled = true; // Habilitar WebXR
        document.body.appendChild(renderer.domElement);

        const labelRenderer = new THREE.CSS2DRenderer();
        labelRenderer.setSize(window.innerWidth, window.innerHeight);
        labelRenderer.domElement.style.position = 'absolute';
        labelRenderer.domElement.style.top = '0px';
        labelRenderer.domElement.style.left = '0px';
        labelRenderer.domElement.style.pointerEvents = 'none';
        document.body.appendChild(labelRenderer.domElement);

        // --- ILUMINACIÓN ---
        const ambientLight = new THREE.AmbientLight(0xffffff, 1.0);
        scene.add(ambientLight);
        const light1 = new THREE.DirectionalLight(0xffeedd, 1.2);
        light1.position.set(1, 1, 1);
        scene.add(light1);
        const light2 = new THREE.DirectionalLight(0x99ccff, 0.8);
        light2.position.set(-1, 0.5, 1);
        scene.add(light2);
        const backLight = new THREE.DirectionalLight(0xccddff, 0.5);
        backLight.position.set(0, 0, -1);
        scene.add(backLight);

        // --- RED DE AYUDA (opcional, visualiza el plano) ---
        const gridHelper = new THREE.GridHelper(5, 20, 0x88ccff, 0x446688);
        gridHelper.visible = false;
        scene.add(gridHelper);

        // --- FUNCIONES DE CREACIÓN DE MODELOS (igual que antes) ---
        function createLabel(text, color = '#ffffff', bg = '#333333') {
            const div = document.createElement('div');
            div.textContent = text;
            div.style.color = color;
            div.style.background = bg;
            div.style.cssText = 'color: white; font-family: Arial; font-size: 18px; font-weight: bold; text-shadow: 2px 2px 4px black; background: rgba(0,0,0,0.8); padding: 6px 14px; border-radius: 20px; border: 1px solid rgba(255,255,255,0.5); white-space: nowrap; backdrop-filter: blur(2px);';
            return new THREE.CSS2DObject(div);
        }

        function createAtom(color, radius, orbitalColor, orbitalRadius = radius * 1.8, orbitalTube = 0.025, nombre = 'Átomo') {
            const group = new THREE.Group();
            const sphereGeo = new THREE.SphereGeometry(radius, 32, 16);
            const sphereMat = new THREE.MeshStandardMaterial({ color, roughness: 0.25, metalness: 0.3, emissive: color, emissiveIntensity: 0.1 });
            const sphere = new THREE.Mesh(sphereGeo, sphereMat);
            group.add(sphere);
            
            const torusGeo = new THREE.TorusGeometry(orbitalRadius, orbitalTube, 16, 32);
            const torusMat = new THREE.MeshStandardMaterial({ color: orbitalColor, emissive: orbitalColor, emissiveIntensity: 0.2, transparent: true, opacity: 0.5, side: THREE.DoubleSide });
            const torus = new THREE.Mesh(torusGeo, torusMat);
            torus.rotation.x = Math.PI / 2;
            torus.rotation.z = 0.3;
            group.add(torus);
            const torus2 = new THREE.Mesh(torusGeo, torusMat);
            torus2.rotation.y = Math.PI / 2;
            torus2.rotation.x = 0.5;
            torus2.scale.set(1.1, 1.1, 1.1);
            group.add(torus2);
            
            const label = createLabel(nombre, '#fff', '#333');
            label.position.y = radius * 2.5;
            group.add(label);
            group.userData.label = label;
            group.userData.type = 'atom';
            return group;
        }

        function createH2() {
            const group = new THREE.Group();
            const hGeo = new THREE.SphereGeometry(0.1, 24, 12);
            const hMat = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0.3, metalness: 0.1, emissive: 0x446688, emissiveIntensity: 0.1 });
            const h1 = new THREE.Mesh(hGeo, hMat);
            h1.position.x = -0.15;
            group.add(h1);
            const h2 = new THREE.Mesh(hGeo, hMat.clone());
            h2.position.x = 0.15;
            group.add(h2);
            const bondGeo = new THREE.CylinderGeometry(0.03, 0.03, 0.3, 6);
            const bondMat = new THREE.MeshStandardMaterial({ color: 0xaaaaaa, roughness: 0.6, metalness: 0.8 });
            const bond = new THREE.Mesh(bondGeo, bondMat);
            bond.rotation.z = Math.PI / 2;
            bond.position.x = 0;
            group.add(bond);
            const label = createLabel('Hidrógeno (H₂)', '#aaddff', '#1a3a4a');
            label.position.y = 0.35;
            group.add(label);
            group.userData.label = label;
            group.userData.type = 'h2';
            return group;
        }

        function createH2O() {
            const group = new THREE.Group();
            const oGeo = new THREE.SphereGeometry(0.16, 32, 16);
            const oMat = new THREE.MeshStandardMaterial({ color: 0xff3333, roughness: 0.25, metalness: 0.2, emissive: 0x331111, emissiveIntensity: 0.2 });
            const oxygen = new THREE.Mesh(oGeo, oMat);
            group.add(oxygen);
            const hGeo = new THREE.SphereGeometry(0.1, 24, 12);
            const hMat = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0.3, metalness: 0.1, emissive: 0x446688, emissiveIntensity: 0.1 });
            const h1 = new THREE.Mesh(hGeo, hMat);
            h1.position.set(0.2, 0.15, 0);
            group.add(h1);
            const h2 = new THREE.Mesh(hGeo, hMat.clone());
            h2.position.set(0.2, -0.15, 0);
            group.add(h2);
            const bondGeo = new THREE.CylinderGeometry(0.035, 0.035, 0.25, 6);
            const bondMat = new THREE.MeshStandardMaterial({ color: 0xcccccc, roughness: 0.5, metalness: 0.8 });
            const bond1 = new THREE.Mesh(bondGeo, bondMat);
            bond1.position.set(0.1, 0.075, 0);
            bond1.rotation.z = -0.7;
            group.add(bond1);
            const bond2 = new THREE.Mesh(bondGeo, bondMat);
            bond2.position.set(0.1, -0.075, 0);
            bond2.rotation.z = 0.7;
            group.add(bond2);
            const label = createLabel('Agua (H₂O)', '#88ddff', '#1a4a5a');
            label.position.y = 0.4;
            group.add(label);
            group.userData.label = label;
            group.userData.type = 'water';
            return group;
        }

        // --- INICIALIZAR MODELOS ---
        function initModels() {
            oxygenModel = createAtom(0xff5555, 0.16, 0x88ccff, 0.28, 0.025, 'Oxígeno (O)');
            oxygenModel.visible = false;
            oxygenModel.userData.label.visible = false;
            scene.add(oxygenModel);

            hydrogenModel = createH2();
            hydrogenModel.visible = false;
            hydrogenModel.userData.label.visible = false;
            scene.add(hydrogenModel);

            waterModel = createH2O();
            waterModel.visible = false;
            waterModel.userData.label.visible = false;
            scene.add(waterModel);
        }
        initModels();

        // --- GESTIÓN DE TOQUES PARA INTERACCIÓN ---
        function setupTouchEvents() {
            const canvas = renderer.domElement;

            canvas.addEventListener('touchstart', (e) => {
                if (e.touches.length === 1) {
                    // Seleccionar objeto para arrastrar
                    const touch = e.touches[0];
                    const coords = new THREE.Vector2(
                        (touch.clientX / window.innerWidth) * 2 - 1,
                        -(touch.clientY / window.innerHeight) * 2 + 1
                    );
                    const raycaster = new THREE.Raycaster();
                    raycaster.setFromCamera(coords, camera);
                    
                    const objects = [oxygenModel, hydrogenModel, waterModel].filter(obj => obj.visible);
                    const intersects = raycaster.intersectObjects(objects, true); // true para hijos
                    
                    if (intersects.length > 0) {
                        // Encontrar el grupo padre (modelo)
                        let obj = intersects[0].object;
                        while (obj.parent && obj.parent !== scene) {
                            obj = obj.parent;
                        }
                        selectedObject = obj;
                        isDragging = true;
                        e.preventDefault();
                    }
                } else if (e.touches.length === 2) {
                    // Iniciar pinch para escalar
                    const dx = e.touches[0].clientX - e.touches[1].clientX;
                    const dy = e.touches[0].clientY - e.touches[1].clientY;
                    initialDistance = Math.sqrt(dx * dx + dy * dy);
                    
                    if (oxygenModel.visible) initialScale = oxygenModel.scale.x;
                    else if (hydrogenModel.visible) initialScale = hydrogenModel.scale.x;
                    else if (waterModel.visible) initialScale = waterModel.scale.x;
                    e.preventDefault();
                }
            }, { passive: false });

            canvas.addEventListener('touchmove', (e) => {
                if (e.touches.length === 1 && isDragging && selectedObject && xrReferenceSpace) {
                    // Mover objeto: convertir toque a rayo y golpear plano Y=0 (suelo)
                    const touch = e.touches[0];
                    const coords = new THREE.Vector2(
                        (touch.clientX / window.innerWidth) * 2 - 1,
                        -(touch.clientY / window.innerHeight) * 2 + 1
                    );
                    const raycaster = new THREE.Raycaster();
                    raycaster.setFromCamera(coords, camera);
                    
                    // Plano en Y=0 (suelo)
                    const plane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
                    const target = new THREE.Vector3();
                    if (raycaster.ray.intersectPlane(plane, target)) {
                        selectedObject.position.copy(target);
                        // Mantener la altura Y (el plano ya da Y=0, ajustar para que quede sobre el suelo)
                        selectedObject.position.y = 0;
                    }
                    e.preventDefault();
                } else if (e.touches.length === 2) {
                    // Pinch para escalar
                    const dx = e.touches[0].clientX - e.touches[1].clientX;
                    const dy = e.touches[0].clientY - e.touches[1].clientY;
                    const distance = Math.sqrt(dx * dx + dy * dy);
                    const scaleFactor = distance / initialDistance;
                    const newScale = initialScale * scaleFactor;
                    
                    if (oxygenModel.visible) oxygenModel.scale.setScalar(newScale);
                    if (hydrogenModel.visible) hydrogenModel.scale.setScalar(newScale);
                    if (waterModel.visible) waterModel.scale.setScalar(newScale);
                    e.preventDefault();
                }
            }, { passive: false });

            canvas.addEventListener('touchend', (e) => {
                if (e.touches.length === 0) {
                    isDragging = false;
                    selectedObject = null;
                }
                
                // Detectar doble tap
                const currentTime = new Date().getTime();
                const tapLength = currentTime - lastTap;
                if (tapLength < 300 && tapLength > 0) {
                    toggleDetailedInfo();
                    e.preventDefault();
                }
                lastTap = currentTime;
            });
        }

        // --- PANEL DE INFORMACIÓN (igual) ---
        let showDetailedInfo = false;
        function toggleDetailedInfo() {
            showDetailedInfo = !showDetailedInfo;
            if (showDetailedInfo) {
                updateInfoPanelContent();
                infoPanel.classList.remove('hidden-panel');
                infoPanel.classList.add('visible-panel');
            } else {
                infoPanel.classList.remove('visible-panel');
                infoPanel.classList.add('hidden-panel');
            }
        }

        function updateInfoPanelContent() {
            if (waterModel.visible) {
                document.getElementById('particle-title').innerText = 'Agua (H₂O)';
                document.getElementById('atomic-number').innerText = '10 (total)';
                document.getElementById('atomic-mass').innerText = '18.015 u';
                document.getElementById('electronegativity').innerText = 'O: 3.44 / H: 2.20';
                document.getElementById('discovery').innerText = 'Conocida desde la antigüedad';
                document.getElementById('extra-info').innerText = 'Molécula polar, disolvente universal.';
            } else if (oxygenModel.visible) {
                document.getElementById('particle-title').innerText = 'Oxígeno (O)';
                document.getElementById('atomic-number').innerText = '8';
                document.getElementById('atomic-mass').innerText = '15.999 u';
                document.getElementById('electronegativity').innerText = '3.44 (Pauling)';
                document.getElementById('discovery').innerText = '1774 - Priestley, Scheele';
                document.getElementById('extra-info').innerText = 'Gas incoloro, esencial para la vida.';
            } else if (hydrogenModel.visible) {
                document.getElementById('particle-title').innerText = 'Hidrógeno (H₂)';
                document.getElementById('atomic-number').innerText = '1';
                document.getElementById('atomic-mass').innerText = '1.008 u';
                document.getElementById('electronegativity').innerText = '2.20';
                document.getElementById('discovery').innerText = '1766 - Henry Cavendish';
                document.getElementById('extra-info').innerText = 'Elemento más abundante del universo.';
            } else {
                document.getElementById('particle-title').innerText = 'Sin objetos';
                document.getElementById('atomic-number').innerText = '--';
                document.getElementById('atomic-mass').innerText = '--';
                document.getElementById('electronegativity').innerText = '--';
                document.getElementById('discovery').innerText = '--';
                document.getElementById('extra-info').innerText = 'Coloca oxígeno e hidrógeno en el suelo.';
            }
        }

        // --- LÓGICA DE REACCIÓN QUÍMICA ---
        function checkFusion() {
            if (oxygenModel.visible && hydrogenModel.visible) {
                const distance = oxygenModel.position.distanceTo(hydrogenModel.position);
                if (distance < 0.3) {
                    // Fusionar
                    if (!waterModel.visible) {
                        // Sonido (opcional)
                        // playSound();
                    }
                    oxygenModel.visible = false;
                    oxygenModel.userData.label.visible = false;
                    hydrogenModel.visible = false;
                    hydrogenModel.userData.label.visible = false;
                    
                    waterModel.position.copy(oxygenModel.position).add(hydrogenModel.position).multiplyScalar(0.5);
                    waterModel.quaternion.copy(oxygenModel.quaternion).slerp(hydrogenModel.quaternion, 0.5);
                    waterModel.scale.setScalar((oxygenModel.scale.x + hydrogenModel.scale.x) / 2);
                    waterModel.visible = true;
                    waterModel.userData.label.visible = true;
                }
            }
            
            // Si no hay fusión, ocultar agua
            if (!oxygenModel.visible || !hydrogenModel.visible || 
                (oxygenModel.visible && hydrogenModel.visible && oxygenModel.position.distanceTo(hydrogenModel.position) >= 0.3)) {
                waterModel.visible = false;
                waterModel.userData.label.visible = false;
            }
        }

        // --- WEBXR: INICIAR SESIÓN AR ---
        async function startAR() {
            if (!navigator.xr) {
                alert('WebXR no está disponible en este navegador. Prueba con Chrome Android.');
                return;
            }

            try {
                const session = await navigator.xr.requestSession('immersive-ar', {
                    requiredFeatures: ['hit-test', 'local-floor']
                });
                xrSession = session;

                // Configurar renderer XR
                await renderer.xr.setSession(session);

                // Referencia espacial
                xrReferenceSpace = await session.requestReferenceSpace('local-floor');

                // Fuente de hit test
                const viewerSpace = await session.requestReferenceSpace('viewer');
                xrHitTestSource = await session.requestHitTestSource({
                    space: viewerSpace
                });

                // Ocultar botón y mostrar estado
                arButton.style.display = 'none';
                statusDiv.innerHTML = '✅ AR iniciado. Toca en una superficie para colocar Oxígeno.';

                // Manejar selección: al tocar, hacer hit test y colocar oxígeno o hidrógeno
                let objectsPlaced = 0;
                const canvas = renderer.domElement;

                canvas.addEventListener('touchstart', async (e) => {
                    if (e.touches.length === 1 && !isDragging) {
                        // Solo si no estamos arrastrando
                        const frame = await new Promise(resolve => {
                            session.requestAnimationFrame((time, frame) => resolve(frame));
                        });
                        
                        if (frame && xrHitTestSource) {
                            const hitTestResults = frame.getHitTestResults(xrHitTestSource);
                            if (hitTestResults.length > 0) {
                                const pose = hitTestResults[0].getPose(xrReferenceSpace);
                                if (pose) {
                                    const pos = new THREE.Vector3(pose.transform.position.x, pose.transform.position.y, pose.transform.position.z);
                                    
                                    if (objectsPlaced === 0) {
                                        oxygenModel.position.copy(pos);
                                        oxygenModel.visible = true;
                                        oxygenModel.userData.label.visible = true;
                                        oxygenModel.scale.setScalar(1);
                                        objectsPlaced++;
                                        statusDiv.innerHTML = '✅ Oxígeno colocado. Toca otra superficie para colocar Hidrógeno.';
                                    } else if (objectsPlaced === 1) {
                                        hydrogenModel.position.copy(pos);
                                        hydrogenModel.visible = true;
                                        hydrogenModel.userData.label.visible = true;
                                        hydrogenModel.scale.setScalar(1);
                                        objectsPlaced++;
                                        statusDiv.innerHTML = '✅ Ambos colocados. Acerca los objetos para la reacción.';
                                    } else {
                                        // Reemplazar el más cercano?
                                        // Por simplicidad, no hacemos nada
                                    }
                                }
                            }
                        }
                    }
                });

                // Loop de animación con hit test continuo (opcional, para debug)
                session.addEventListener('frame', (time, frame) => {
                    // No necesario para la lógica actual
                });

            } catch (e) {
                console.error(e);
                alert('Error al iniciar AR: ' + e.message);
            }
        }

        // --- BOTÓN DE INICIO ---
        arButton.addEventListener('click', startAR);

        // --- CONFIGURAR EVENTOS TÁCTILES ---
        setupTouchEvents();

        // --- LOOP DE RENDER ---
        function animate() {
            // Verificar fusión en cada frame
            checkFusion();
            
            renderer.render(scene, camera);
            labelRenderer.render(scene, camera);
            
            if (xrSession) {
                // El loop lo maneja WebXR
                xrSession.requestAnimationFrame(animate);
            } else {
                requestAnimationFrame(animate);
            }
        }
        animate();

        // --- REDIMENSIONAR ---
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
            labelRenderer.setSize(window.innerWidth, window.innerHeight);
        });

        // Inicializar panel oculto
        infoPanel.classList.add('hidden-panel');
    </script>
</body>
</html>
    <script>
console.log('Hello World!');
    </script>
</body>
</html>
