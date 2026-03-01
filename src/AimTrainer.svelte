<script>
    import { onMount, onDestroy, createEventDispatcher } from "svelte";
    import * as THREE from "three";

    const dispatch = createEventDispatcher();

    // Phase state: loading -> nickname -> menu -> playing -> results
    let phase = "loading"; // loading | nickname | menu | playing | results
    let loadProgress = 0;
    let gameMode = "static";
    let score = 0;
    let hits = 0;
    let misses = 0;
    let timeLeft = 30;
    let sensitivity = 2.5;
    let timerInterval;

    // Nickname & Leaderboard
    let nickname = "";
    let nicknameInput = "";
    let leaderboard = [];
    const LB_KEY = "aimtrainer_leaderboard";
    const NICK_KEY = "aimtrainer_nickname";

    function loadNickname() {
        const saved = localStorage.getItem(NICK_KEY);
        if (saved) {
            nickname = saved;
            return true;
        }
        return false;
    }

    function saveNickname(name) {
        nickname = name.trim().slice(0, 16);
        localStorage.setItem(NICK_KEY, nickname);
    }

    function loadLeaderboard() {
        try {
            const data = JSON.parse(localStorage.getItem(LB_KEY) || "[]");
            leaderboard = data.sort((a, b) => b.score - a.score).slice(0, 10);
        } catch {
            leaderboard = [];
        }
    }

    function saveScore(playerName, playerScore, playerAccuracy, playerMode) {
        loadLeaderboard();
        leaderboard.push({
            name: playerName,
            score: playerScore,
            accuracy: playerAccuracy,
            mode: playerMode,
            date: new Date().toLocaleDateString("ru-RU"),
        });
        leaderboard = leaderboard
            .sort((a, b) => b.score - a.score)
            .slice(0, 10);
        localStorage.setItem(LB_KEY, JSON.stringify(leaderboard));
    }

    function confirmNickname() {
        if (!nicknameInput.trim()) return;
        saveNickname(nicknameInput);
        phase = "menu";
    }

    function handleNicknameKey(e) {
        if (e.key === "Enter") confirmNickname();
    }

    // Three.js
    let container;
    let renderer, scene, camera;
    let targets = [];
    let animationId;
    let clock;

    // Gun
    let gunGroup;
    let gunRecoil = 0;
    let muzzleFlash;

    // Tracers
    let tracers = [];

    // Camera
    let yaw = 0;
    let pitch = 0;
    let isLocked = false;

    // Effects
    let hitFlash = false;
    let missFlash = false;
    let comboCount = 0;
    let bestCombo = 0;
    let lastHitTime = 0;
    let floatingTexts = [];
    let floatingTextId = 0;

    // Config
    const TARGET_COUNT_STATIC = 5;
    const TARGET_COUNT_MOVING = 3;
    const ARENA_SIZE = 20;
    const TARGET_RADIUS = 0.5;

    $: accuracy =
        hits + misses > 0 ? Math.round((hits / (hits + misses)) * 100) : 0;

    function close() {
        cleanup();
        dispatch("close");
    }

    function cleanup() {
        if (animationId) cancelAnimationFrame(animationId);
        if (timerInterval) clearInterval(timerInterval);
        if (renderer) {
            renderer.dispose();
            renderer.forceContextLoss();
        }
        document.exitPointerLock?.();
        isLocked = false;
    }

    // ======================== LOADING ========================
    function startLoading() {
        phase = "loading";
        loadProgress = 0;
        const duration = 2200;
        const steps = 80;
        const stepTime = duration / steps;
        let step = 0;

        const interval = setInterval(() => {
            step++;
            const t = step / steps;
            loadProgress = Math.min(100, (1 - Math.pow(1 - t, 2.2)) * 100);
            if (step >= steps) {
                clearInterval(interval);
                setTimeout(() => {
                    loadLeaderboard();
                    if (loadNickname()) {
                        phase = "menu";
                    } else {
                        phase = "nickname";
                    }
                }, 200);
            }
        }, stepTime);
    }

    // ======================== THREE.JS SCENE ========================
    function initScene() {
        renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        renderer.shadowMap.enabled = true;
        renderer.toneMapping = THREE.ACESFilmicToneMapping;
        renderer.toneMappingExposure = 1.2;
        container.appendChild(renderer.domElement);

        scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0a0a12);
        scene.fog = new THREE.Fog(0x0a0a12, 15, 45);

        camera = new THREE.PerspectiveCamera(
            75,
            window.innerWidth / window.innerHeight,
            0.1,
            100,
        );
        camera.position.set(0, 1.7, 0);
        camera.rotation.order = "YXZ";

        clock = new THREE.Clock();

        buildArena();
        buildGun();

        // Lights
        scene.add(new THREE.AmbientLight(0x1a1a2e, 0.6));
        const mainLight = new THREE.PointLight(0x6366f1, 2, 30);
        mainLight.position.set(0, 8, 0);
        mainLight.castShadow = true;
        scene.add(mainLight);

        const accent1 = new THREE.PointLight(0xf43f5e, 1.5, 20);
        accent1.position.set(-8, 4, -8);
        scene.add(accent1);

        const accent2 = new THREE.PointLight(0x06b6d4, 1.5, 20);
        accent2.position.set(8, 4, 8);
        scene.add(accent2);

        window.addEventListener("resize", handleResize);
    }

    function buildArena() {
        // Floor
        const floorGeo = new THREE.PlaneGeometry(
            ARENA_SIZE * 2,
            ARENA_SIZE * 2,
        );
        const floorMat = new THREE.MeshStandardMaterial({
            color: 0x0a0a12,
            roughness: 0.8,
            metalness: 0.2,
        });
        const floor = new THREE.Mesh(floorGeo, floorMat);
        floor.rotation.x = -Math.PI / 2;
        floor.receiveShadow = true;
        scene.add(floor);

        // Grid
        const grid = new THREE.GridHelper(
            ARENA_SIZE * 2,
            40,
            0x1e1e3f,
            0x1e1e3f,
        );
        grid.position.y = 0.01;
        scene.add(grid);

        // Walls
        const wallMat = new THREE.MeshStandardMaterial({
            color: 0x12121f,
            roughness: 0.9,
            metalness: 0.1,
            transparent: true,
            opacity: 0.6,
        });

        const backWall = new THREE.Mesh(
            new THREE.PlaneGeometry(ARENA_SIZE * 2, 10),
            wallMat,
        );
        backWall.position.set(0, 5, -ARENA_SIZE);
        scene.add(backWall);

        const leftWall = new THREE.Mesh(
            new THREE.PlaneGeometry(ARENA_SIZE * 2, 10),
            wallMat,
        );
        leftWall.position.set(-ARENA_SIZE, 5, 0);
        leftWall.rotation.y = Math.PI / 2;
        scene.add(leftWall);

        const rightWall = new THREE.Mesh(
            new THREE.PlaneGeometry(ARENA_SIZE * 2, 10),
            wallMat,
        );
        rightWall.position.set(ARENA_SIZE, 5, 0);
        rightWall.rotation.y = -Math.PI / 2;
        scene.add(rightWall);

        // Neon edge lines
        const edgeMat = new THREE.LineBasicMaterial({ color: 0x6366f1 });
        [
            [
                [-ARENA_SIZE, 0, -ARENA_SIZE],
                [ARENA_SIZE, 0, -ARENA_SIZE],
            ],
            [
                [-ARENA_SIZE, 0, -ARENA_SIZE],
                [-ARENA_SIZE, 0, ARENA_SIZE],
            ],
            [
                [ARENA_SIZE, 0, -ARENA_SIZE],
                [ARENA_SIZE, 0, ARENA_SIZE],
            ],
            [
                [-ARENA_SIZE, 10, -ARENA_SIZE],
                [ARENA_SIZE, 10, -ARENA_SIZE],
            ],
        ].forEach(([a, b]) => {
            const geo = new THREE.BufferGeometry().setFromPoints([
                new THREE.Vector3(...a),
                new THREE.Vector3(...b),
            ]);
            scene.add(new THREE.Line(geo, edgeMat));
        });

        // Floating particles
        const pGeo = new THREE.BufferGeometry();
        const pCount = 200;
        const pPos = new Float32Array(pCount * 3);
        for (let i = 0; i < pCount; i++) {
            pPos[i * 3] = (Math.random() - 0.5) * ARENA_SIZE * 2;
            pPos[i * 3 + 1] = Math.random() * 8;
            pPos[i * 3 + 2] = (Math.random() - 0.5) * ARENA_SIZE * 2;
        }
        pGeo.setAttribute("position", new THREE.BufferAttribute(pPos, 3));
        scene.add(
            new THREE.Points(
                pGeo,
                new THREE.PointsMaterial({
                    color: 0x6366f1,
                    size: 0.05,
                    transparent: true,
                    opacity: 0.6,
                }),
            ),
        );
    }

    function buildGun() {
        gunGroup = new THREE.Group();

        // Pistol body — blocky low-poly style
        const bodyMat = new THREE.MeshStandardMaterial({
            color: 0x2a2a35,
            roughness: 0.4,
            metalness: 0.8,
        });
        const accentMat = new THREE.MeshStandardMaterial({
            color: 0x6366f1,
            emissive: 0x6366f1,
            emissiveIntensity: 0.3,
            roughness: 0.3,
            metalness: 0.9,
        });

        // Slide (top part)
        const slide = new THREE.Mesh(
            new THREE.BoxGeometry(0.06, 0.06, 0.35),
            bodyMat,
        );
        slide.position.set(0, 0.03, -0.05);
        gunGroup.add(slide);

        // Barrel
        const barrel = new THREE.Mesh(
            new THREE.CylinderGeometry(0.015, 0.015, 0.12, 8),
            bodyMat,
        );
        barrel.rotation.x = Math.PI / 2;
        barrel.position.set(0, 0.03, -0.28);
        gunGroup.add(barrel);

        // Frame (lower part)
        const frame = new THREE.Mesh(
            new THREE.BoxGeometry(0.055, 0.04, 0.25),
            bodyMat,
        );
        frame.position.set(0, -0.01, 0.0);
        gunGroup.add(frame);

        // Grip
        const grip = new THREE.Mesh(
            new THREE.BoxGeometry(0.05, 0.12, 0.06),
            bodyMat,
        );
        grip.position.set(0, -0.07, 0.08);
        grip.rotation.x = 0.2;
        gunGroup.add(grip);

        // Trigger guard
        const trigGuard = new THREE.Mesh(
            new THREE.TorusGeometry(0.025, 0.005, 6, 8, Math.PI),
            accentMat,
        );
        trigGuard.position.set(0, -0.03, 0.03);
        trigGuard.rotation.y = Math.PI / 2;
        gunGroup.add(trigGuard);

        // Accent stripe on slide
        const stripe = new THREE.Mesh(
            new THREE.BoxGeometry(0.065, 0.008, 0.12),
            accentMat,
        );
        stripe.position.set(0, 0.065, -0.1);
        gunGroup.add(stripe);

        // Muzzle flash (hidden by default)
        const flashGeo = new THREE.SphereGeometry(0.04, 8, 8);
        const flashMat = new THREE.MeshBasicMaterial({
            color: 0xfbbf24,
            transparent: true,
            opacity: 0,
        });
        muzzleFlash = new THREE.Mesh(flashGeo, flashMat);
        muzzleFlash.position.set(0, 0.03, -0.34);
        gunGroup.add(muzzleFlash);

        // Position gun in bottom-right of camera view
        gunGroup.position.set(0.25, -0.22, -0.4);
        gunGroup.rotation.set(0, 0, 0);

        camera.add(gunGroup);
        scene.add(camera);
    }

    // ======================== TARGETS ========================
    function spawnTargets() {
        targets.forEach((t) => {
            scene.remove(t.mesh);
            scene.remove(t.glow);
        });
        targets = [];
        const count =
            gameMode === "static" ? TARGET_COUNT_STATIC : TARGET_COUNT_MOVING;
        for (let i = 0; i < count; i++) spawnSingleTarget();
    }

    function spawnSingleTarget(replaceIndex = -1) {
        const geo = new THREE.SphereGeometry(TARGET_RADIUS, 32, 32);
        const mat = new THREE.MeshStandardMaterial({
            color: 0xf43f5e,
            emissive: 0xf43f5e,
            emissiveIntensity: 0.4,
            roughness: 0.3,
            metalness: 0.7,
        });
        const mesh = new THREE.Mesh(geo, mat);

        const x = (Math.random() - 0.5) * 16;
        const y = 1 + Math.random() * 4;
        const z = -3 - Math.random() * 12;
        mesh.position.set(x, y, z);
        mesh.castShadow = true;
        scene.add(mesh);

        const glowGeo = new THREE.SphereGeometry(TARGET_RADIUS * 1.6, 16, 16);
        const glowMat = new THREE.MeshBasicMaterial({
            color: 0xf43f5e,
            transparent: true,
            opacity: 0.15,
        });
        const glow = new THREE.Mesh(glowGeo, glowMat);
        glow.position.copy(mesh.position);
        scene.add(glow);

        const target = {
            mesh,
            glow,
            baseX: x,
            moveSpeed: 1 + Math.random() * 2,
            moveRange: 2 + Math.random() * 3,
            movePhase: Math.random() * Math.PI * 2,
            spawnTime: clock.getElapsedTime(),
        };

        if (replaceIndex >= 0) targets[replaceIndex] = target;
        else targets.push(target);
    }

    // ======================== SHOOTING ========================
    function shoot() {
        if (phase !== "playing") return;

        // Gun recoil animation
        gunRecoil = 1;

        // Muzzle flash
        muzzleFlash.material.opacity = 1;
        setTimeout(() => {
            muzzleFlash.material.opacity = 0;
        }, 60);

        // Raycaster
        const raycaster = new THREE.Raycaster();
        raycaster.setFromCamera(new THREE.Vector2(0, 0), camera);

        // Create tracer line
        const origin = new THREE.Vector3();
        camera.getWorldPosition(origin);
        const dir = new THREE.Vector3();
        camera.getWorldDirection(dir);
        const tracerEnd = origin.clone().add(dir.multiplyScalar(50));
        createTracer(origin, tracerEnd);

        const meshes = targets.map((t) => t.mesh);
        const intersects = raycaster.intersectObjects(meshes);

        if (intersects.length > 0) {
            const hitMesh = intersects[0].object;
            const idx = targets.findIndex((t) => t.mesh === hitMesh);
            if (idx !== -1) handleHit(idx, intersects[0].point);
        } else {
            misses++;
            comboCount = 0;
            missFlash = true;
            setTimeout(() => (missFlash = false), 100);
        }
    }

    function createTracer(start, end) {
        const geo = new THREE.BufferGeometry().setFromPoints([start, end]);
        const mat = new THREE.LineBasicMaterial({
            color: 0xfbbf24,
            transparent: true,
            opacity: 0.8,
        });
        const line = new THREE.Line(geo, mat);
        scene.add(line);
        tracers.push({ line, mat, life: 1 });
    }

    function handleHit(targetIndex, hitPoint) {
        const now = Date.now();
        const dt = now - lastHitTime;
        lastHitTime = now;

        comboCount = dt < 1500 ? comboCount + 1 : 1;
        if (comboCount > bestCombo) bestCombo = comboCount;

        const mul = Math.min(comboCount, 5);
        const points = 100 * mul;
        score += points;
        hits++;

        hitFlash = true;
        setTimeout(() => (hitFlash = false), 80);
        addFloatingText(
            `+${points}`,
            comboCount > 1 ? `${comboCount}x COMBO` : "",
        );

        const t = targets[targetIndex];
        scene.remove(t.mesh);
        scene.remove(t.glow);
        t.mesh.geometry.dispose();
        t.mesh.material.dispose();
        t.glow.geometry.dispose();
        t.glow.material.dispose();

        createHitParticles(hitPoint);
        spawnSingleTarget(targetIndex);
    }

    function createHitParticles(point) {
        const cnt = 15;
        const geo = new THREE.BufferGeometry();
        const pos = new Float32Array(cnt * 3);
        const vels = [];
        for (let i = 0; i < cnt; i++) {
            pos[i * 3] = point.x;
            pos[i * 3 + 1] = point.y;
            pos[i * 3 + 2] = point.z;
            vels.push(
                new THREE.Vector3(
                    (Math.random() - 0.5) * 4,
                    (Math.random() - 0.5) * 4,
                    (Math.random() - 0.5) * 4,
                ),
            );
        }
        geo.setAttribute("position", new THREE.BufferAttribute(pos, 3));
        const mat = new THREE.PointsMaterial({
            color: 0xfbbf24,
            size: 0.12,
            transparent: true,
            opacity: 1,
        });
        const pts = new THREE.Points(geo, mat);
        scene.add(pts);

        let life = 1;
        function anim() {
            life -= 0.04;
            if (life <= 0) {
                scene.remove(pts);
                geo.dispose();
                mat.dispose();
                return;
            }
            mat.opacity = life;
            const p = geo.attributes.position.array;
            for (let i = 0; i < cnt; i++) {
                p[i * 3] += vels[i].x * 0.05;
                p[i * 3 + 1] += vels[i].y * 0.05;
                p[i * 3 + 2] += vels[i].z * 0.05;
            }
            geo.attributes.position.needsUpdate = true;
            requestAnimationFrame(anim);
        }
        anim();
    }

    function addFloatingText(main, sub) {
        const id = floatingTextId++;
        floatingTexts = [...floatingTexts, { id, main, sub }];
        setTimeout(() => {
            floatingTexts = floatingTexts.filter((t) => t.id !== id);
        }, 800);
    }

    // ======================== GAME FLOW ========================
    function startGame() {
        score = 0;
        hits = 0;
        misses = 0;
        timeLeft = 30;
        comboCount = 0;
        bestCombo = 0;
        lastHitTime = 0;
        phase = "playing";
        yaw = 0;
        pitch = 0;
        camera.rotation.set(0, 0, 0);
        spawnTargets();
        container.requestPointerLock?.();
        timerInterval = setInterval(() => {
            timeLeft--;
            if (timeLeft <= 0) endGame();
        }, 1000);
    }

    function endGame() {
        phase = "results";
        clearInterval(timerInterval);
        document.exitPointerLock?.();
        isLocked = false;
        targets.forEach((t) => {
            scene.remove(t.mesh);
            scene.remove(t.glow);
        });
        targets = [];
        // Save to leaderboard
        if (score > 0) saveScore(nickname, score, accuracy, gameMode);
    }

    function backToMenu() {
        phase = "menu";
    }

    // ======================== INPUT ========================
    function handleMouseMove(e) {
        if (!isLocked || phase !== "playing") return;
        yaw -= e.movementX * sensitivity * 0.001;
        pitch -= e.movementY * sensitivity * 0.001;
        pitch = Math.max(-Math.PI / 2.2, Math.min(Math.PI / 2.2, pitch));
        camera.rotation.y = yaw;
        camera.rotation.x = pitch;
    }

    function handleMouseDown(e) {
        if (e.button === 0 && phase === "playing" && isLocked) shoot();
    }

    function handlePointerLockChange() {
        isLocked = document.pointerLockElement === container;
    }

    function handleKeyDown(e) {
        if (e.key === "Escape" && phase === "playing") endGame();
    }

    function handleResize() {
        if (!renderer || !camera) return;
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    }

    // ======================== RENDER LOOP ========================
    function animate() {
        animationId = requestAnimationFrame(animate);
        if (!renderer || !scene || !camera) return;
        const elapsed = clock.getElapsedTime();

        // Moving targets
        if (phase === "playing" && gameMode === "moving") {
            targets.forEach((t) => {
                const nx =
                    t.baseX +
                    Math.sin(elapsed * t.moveSpeed + t.movePhase) * t.moveRange;
                t.mesh.position.x = nx;
                t.glow.position.x = nx;
            });
        }

        // Pulse glow
        targets.forEach((t) => {
            t.glow.material.opacity =
                0.12 + Math.sin(elapsed * 3 + t.movePhase) * 0.05;
            t.glow.scale.setScalar(
                1 + Math.sin(elapsed * 2 + t.movePhase) * 0.06,
            );
        });

        // Gun recoil spring-back
        if (gunGroup) {
            gunRecoil *= 0.82;
            gunGroup.position.z = -0.4 + gunRecoil * 0.06;
            gunGroup.rotation.x = -gunRecoil * 0.15;
        }

        // Update tracers (fade out)
        tracers = tracers.filter((tr) => {
            tr.life -= 0.08;
            if (tr.life <= 0) {
                scene.remove(tr.line);
                tr.line.geometry.dispose();
                tr.mat.dispose();
                return false;
            }
            tr.mat.opacity = tr.life * 0.8;
            return true;
        });

        renderer.render(scene, camera);
    }

    onMount(() => {
        initScene();
        animate();
        startLoading();

        document.addEventListener("mousemove", handleMouseMove);
        document.addEventListener("mousedown", handleMouseDown);
        document.addEventListener("pointerlockchange", handlePointerLockChange);
        document.addEventListener("keydown", handleKeyDown);
    });

    onDestroy(() => {
        cleanup();
        document.removeEventListener("mousemove", handleMouseMove);
        document.removeEventListener("mousedown", handleMouseDown);
        document.removeEventListener(
            "pointerlockchange",
            handlePointerLockChange,
        );
        document.removeEventListener("keydown", handleKeyDown);
    });
</script>

<!-- ===================== TEMPLATE ===================== -->
<div class="aim-overlay">
    <!-- Window title bar (macOS style) -->
    <div class="aim-titlebar">
        <div class="traffic-lights">
            <button class="dot red" on:click={close}></button>
            <button class="dot yellow" on:click={close}></button>
            <button class="dot green"></button>
        </div>
        <div class="titlebar-center">
            <span class="titlebar-text">aim.trainer</span>
        </div>
        <div class="titlebar-spacer"></div>
    </div>
    <div class="titlebar-line"></div>

    <div class="game-container" bind:this={container}>
        <!-- ===== LOADING ===== -->
        {#if phase === "loading"}
            <div class="loading-screen">
                <div class="loading-content">
                    <div class="loading-icon">🎯</div>
                    <span class="loading-brand">aim trainer</span>
                    <span class="loading-sub">by haos1y</span>
                    <div class="loading-bar-wrap">
                        <div class="loading-bar-track">
                            <div
                                class="loading-bar-fill"
                                style="width: {loadProgress}%"
                            ></div>
                        </div>
                        <span class="loading-pct"
                            >{Math.floor(loadProgress)}%</span
                        >
                    </div>
                </div>
            </div>
        {/if}

        <!-- ===== NICKNAME INPUT ===== -->
        {#if phase === "nickname"}
            <div class="menu-screen">
                <div class="menu-window">
                    <div class="menu-titlebar">
                        <div class="menu-tl">
                            <span class="mtl-dot mtl-red"></span>
                            <span class="mtl-dot mtl-yellow"></span>
                            <span class="mtl-dot mtl-green"></span>
                        </div>
                        <span class="menu-titlebar-text">добро пожаловать</span>
                        <div class="menu-tl-spacer"></div>
                    </div>
                    <div class="menu-titlebar-line"></div>
                    <div class="menu-body nick-body">
                        <div class="nick-icon">🎯</div>
                        <h2 class="nick-title">AIM TRAINER</h2>
                        <p class="nick-sub">
                            Введи никнейм для таблицы лидеров
                        </p>
                        <input
                            type="text"
                            class="nick-input"
                            placeholder="Никнейм..."
                            maxlength="16"
                            bind:value={nicknameInput}
                            on:keydown={handleNicknameKey}
                        />
                        <button
                            class="play-btn"
                            on:click={confirmNickname}
                            disabled={!nicknameInput.trim()}
                        >
                            <span>ПРОДОЛЖИТЬ</span>
                        </button>
                    </div>
                </div>
            </div>
        {/if}

        <!-- ===== MENU ===== -->
        {#if phase === "menu"}
            <div class="menu-screen">
                <div class="menu-window menu-window-wide">
                    <div class="menu-titlebar">
                        <div class="menu-tl">
                            <span class="mtl-dot mtl-red"></span>
                            <span class="mtl-dot mtl-yellow"></span>
                            <span class="mtl-dot mtl-green"></span>
                        </div>
                        <span class="menu-titlebar-text"
                            >aim.trainer — {nickname}</span
                        >
                        <div class="menu-tl-spacer"></div>
                    </div>
                    <div class="menu-titlebar-line"></div>

                    <div class="menu-columns">
                        <!-- Left: Settings -->
                        <div class="menu-body">
                            <h2 class="menu-heading">Режим игры</h2>

                            <div class="mode-select">
                                <button
                                    class="mode-btn"
                                    class:active={gameMode === "static"}
                                    on:click={() => (gameMode = "static")}
                                >
                                    <svg
                                        width="20"
                                        height="20"
                                        viewBox="0 0 24 24"
                                        fill="none"
                                        stroke="currentColor"
                                        stroke-width="1.5"
                                    >
                                        <circle cx="12" cy="12" r="10" /><circle
                                            cx="12"
                                            cy="12"
                                            r="4"
                                        /><circle cx="12" cy="12" r="1" />
                                    </svg>
                                    <div class="mode-info">
                                        <span class="mode-name">Static</span>
                                        <span class="mode-desc"
                                            >Неподвижные цели</span
                                        >
                                    </div>
                                </button>
                                <button
                                    class="mode-btn"
                                    class:active={gameMode === "moving"}
                                    on:click={() => (gameMode = "moving")}
                                >
                                    <svg
                                        width="20"
                                        height="20"
                                        viewBox="0 0 24 24"
                                        fill="none"
                                        stroke="currentColor"
                                        stroke-width="1.5"
                                    >
                                        <path d="M5 12h14M12 5l7 7-7 7" />
                                    </svg>
                                    <div class="mode-info">
                                        <span class="mode-name">Moving</span>
                                        <span class="mode-desc"
                                            >Цели двигаются</span
                                        >
                                    </div>
                                </button>
                            </div>

                            <div class="setting-row">
                                <span class="setting-label"
                                    >Чувствительность</span
                                >
                                <span class="setting-val"
                                    >{sensitivity.toFixed(1)}</span
                                >
                            </div>
                            <input
                                type="range"
                                min="0.5"
                                max="5"
                                step="0.1"
                                bind:value={sensitivity}
                                class="sens-slider"
                            />

                            <button class="play-btn" on:click={startGame}>
                                <svg
                                    width="16"
                                    height="16"
                                    viewBox="0 0 24 24"
                                    fill="currentColor"
                                    ><path d="M8 5v14l11-7z" /></svg
                                >
                                <span>PLAY</span>
                            </button>

                            <div class="menu-hints">
                                <span
                                    ><span class="kbd">LMB</span> Стрелять</span
                                >
                                <span><span class="kbd">ESC</span> Выйти</span>
                            </div>
                        </div>

                        <!-- Right: Leaderboard -->
                        <div class="lb-panel">
                            <h2 class="menu-heading">🏆 Топ 10</h2>
                            {#if leaderboard.length === 0}
                                <div class="lb-empty">Нет результатов</div>
                            {:else}
                                <div class="lb-table">
                                    <div class="lb-header">
                                        <span class="lb-rank">#</span>
                                        <span class="lb-name">Игрок</span>
                                        <span class="lb-score">Очки</span>
                                        <span class="lb-acc">%</span>
                                    </div>
                                    {#each leaderboard as entry, i}
                                        <div
                                            class="lb-row"
                                            class:lb-gold={i === 0}
                                            class:lb-silver={i === 1}
                                            class:lb-bronze={i === 2}
                                            class:lb-me={entry.name ===
                                                nickname}
                                        >
                                            <span class="lb-rank"
                                                >{i === 0
                                                    ? "🥇"
                                                    : i === 1
                                                      ? "🥈"
                                                      : i === 2
                                                        ? "🥉"
                                                        : i + 1}</span
                                            >
                                            <span class="lb-name"
                                                >{entry.name}</span
                                            >
                                            <span class="lb-score"
                                                >{entry.score}</span
                                            >
                                            <span class="lb-acc"
                                                >{entry.accuracy}%</span
                                            >
                                        </div>
                                    {/each}
                                </div>
                            {/if}
                        </div>
                    </div>
                </div>
            </div>
        {/if}

        <!-- ===== PLAYING HUD ===== -->
        {#if phase === "playing"}
            <div class="hud">
                <div class="hud-top">
                    <div class="hud-stat">
                        <span class="hud-label">SCORE</span>
                        <span class="hud-value score-val">{score}</span>
                    </div>
                    <div class="hud-timer" class:urgent={timeLeft <= 5}>
                        <span class="timer-big">{timeLeft}</span>
                        <span class="timer-label">sec</span>
                    </div>
                    <div class="hud-stat">
                        <span class="hud-label">ACCURACY</span>
                        <span class="hud-value">{accuracy}%</span>
                    </div>
                </div>
                {#if comboCount > 1}
                    <div class="combo-pop">
                        <span class="combo-num">{comboCount}x</span>
                        <span class="combo-word">COMBO</span>
                    </div>
                {/if}
            </div>

            <!-- Crosshair -->
            <div class="crosshair" class:hit={hitFlash} class:miss={missFlash}>
                <div class="ch-l ch-t"></div>
                <div class="ch-l ch-r"></div>
                <div class="ch-l ch-b"></div>
                <div class="ch-l ch-le"></div>
                <div class="ch-dot"></div>
            </div>

            {#if hitFlash}<div class="hit-overlay"></div>{/if}

            {#each floatingTexts as ft (ft.id)}
                <div class="float-txt">
                    <span class="ft-m">{ft.main}</span>
                    {#if ft.sub}<span class="ft-s">{ft.sub}</span>{/if}
                </div>
            {/each}

            <div class="mode-tag">
                {gameMode === "static" ? "⊚ STATIC" : "↔ MOVING"}
            </div>
        {/if}

        <!-- ===== RESULTS ===== -->
        {#if phase === "results"}
            <div class="results-screen">
                <div class="results-window">
                    <div class="results-titlebar">
                        <div class="menu-tl">
                            <span class="mtl-dot mtl-red"></span>
                            <span class="mtl-dot mtl-yellow"></span>
                            <span class="mtl-dot mtl-green"></span>
                        </div>
                        <span class="menu-titlebar-text">результаты</span>
                        <div class="menu-tl-spacer"></div>
                    </div>
                    <div class="menu-titlebar-line"></div>

                    <div class="results-body">
                        <div class="results-grid">
                            <div class="r-item big">
                                <span class="r-val">{score}</span>
                                <span class="r-lbl">ОЧКИ</span>
                            </div>
                            <div class="r-item">
                                <span class="r-val">{accuracy}%</span>
                                <span class="r-lbl">ТОЧНОСТЬ</span>
                            </div>
                            <div class="r-item">
                                <span class="r-val">{hits}</span>
                                <span class="r-lbl">ПОПАДАНИЙ</span>
                            </div>
                            <div class="r-item">
                                <span class="r-val">{misses}</span>
                                <span class="r-lbl">ПРОМАХОВ</span>
                            </div>
                            <div class="r-item">
                                <span class="r-val">{bestCombo}x</span>
                                <span class="r-lbl">МАКС КОМБО</span>
                            </div>
                            <div class="r-item">
                                <span class="r-val"
                                    >{hits > 0
                                        ? (30 / hits).toFixed(1)
                                        : "—"}s</span
                                >
                                <span class="r-lbl">СР. ВРЕМЯ</span>
                            </div>
                        </div>

                        <!-- Leaderboard in results -->
                        {#if leaderboard.length > 0}
                            <div class="results-lb">
                                <h3 class="results-lb-title">
                                    🏆 Таблица лидеров
                                </h3>
                                <div class="lb-table lb-compact">
                                    {#each leaderboard.slice(0, 5) as entry, i}
                                        <div
                                            class="lb-row"
                                            class:lb-gold={i === 0}
                                            class:lb-silver={i === 1}
                                            class:lb-bronze={i === 2}
                                            class:lb-me={entry.name ===
                                                nickname}
                                        >
                                            <span class="lb-rank"
                                                >{i === 0
                                                    ? "🥇"
                                                    : i === 1
                                                      ? "🥈"
                                                      : i === 2
                                                        ? "🥉"
                                                        : i + 1}</span
                                            >
                                            <span class="lb-name"
                                                >{entry.name}</span
                                            >
                                            <span class="lb-score"
                                                >{entry.score}</span
                                            >
                                        </div>
                                    {/each}
                                </div>
                            </div>
                        {/if}

                        <div class="results-btns">
                            <button class="rbtn primary" on:click={startGame}>
                                <svg
                                    width="16"
                                    height="16"
                                    viewBox="0 0 24 24"
                                    fill="none"
                                    stroke="currentColor"
                                    stroke-width="2"
                                    ><path
                                        d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"
                                    /><path d="M3 3v5h5" /></svg
                                >
                                ЗАНОВО
                            </button>
                            <button class="rbtn" on:click={backToMenu}
                                >МЕНЮ</button
                            >
                            <button class="rbtn" on:click={close}>ВЫХОД</button>
                        </div>
                    </div>
                </div>
            </div>
        {/if}
    </div>
</div>

<!-- ===================== STYLES ===================== -->
<style>
    /* ── OVERLAY + WINDOW FRAME ── */
    .aim-overlay {
        position: fixed;
        inset: 0;
        z-index: 9000;
        background: #000;
        animation: overlayIn 0.4s cubic-bezier(0.16, 1, 0.3, 1);
        display: flex;
        flex-direction: column;
    }

    @keyframes overlayIn {
        from {
            opacity: 0;
            transform: scale(0.96);
        }
        to {
            opacity: 1;
            transform: scale(1);
        }
    }

    /* ── TITLE BAR (macOS) ── */
    .aim-titlebar {
        display: flex;
        align-items: center;
        height: 28px;
        background: rgba(32, 32, 32, 0.95);
        flex-shrink: 0;
        z-index: 50;
    }

    .traffic-lights {
        display: flex;
        align-items: center;
        gap: 8px;
        padding: 0 12px;
        width: 68px;
        height: 12px;
    }

    .dot {
        width: 12px;
        height: 12px;
        border: 0.5px solid #666;
        border-radius: 50%;
        cursor: pointer;
        transition: all 0.15s;
        position: relative;
    }
    .dot::before {
        content: "";
        position: absolute;
        inset: -6px;
    }
    .dot:hover {
        transform: scale(1.15);
        filter: brightness(1.1);
    }
    .dot:active {
        transform: scale(0.95);
    }
    .red {
        background: #ff5f57;
    }
    .yellow {
        background: #febc2e;
    }
    .green {
        background: #28c840;
    }

    .titlebar-center {
        flex: 1;
        text-align: center;
        margin-right: 68px;
    }

    .titlebar-text {
        font-family:
            "SF Pro Display",
            -apple-system,
            BlinkMacSystemFont,
            sans-serif;
        font-size: 13px;
        font-weight: 500;
        color: rgba(255, 255, 255, 0.5);
    }

    .titlebar-spacer {
        width: 0;
    }

    .titlebar-line {
        width: 100%;
        height: 1px;
        background: #2a2a2a;
        flex-shrink: 0;
    }

    /* ── GAME CONTAINER ── */
    .game-container {
        flex: 1;
        position: relative;
        overflow: hidden;
        cursor: none;
    }

    .game-container :global(canvas) {
        display: block;
        width: 100%;
        height: 100%;
    }

    /* ── LOADING SCREEN ── */
    .loading-screen {
        position: absolute;
        inset: 0;
        background: #000;
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 30;
    }

    .loading-content {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 12px;
        animation: loadIn 0.4s cubic-bezier(0.16, 1, 0.3, 1);
    }

    @keyframes loadIn {
        from {
            opacity: 0;
            transform: translateY(10px);
        }
        to {
            opacity: 1;
            transform: translateY(0);
        }
    }

    .loading-icon {
        font-size: 48px;
    }

    .loading-brand {
        font-family:
            "SF Pro Display",
            -apple-system,
            BlinkMacSystemFont,
            sans-serif;
        font-size: 28px;
        font-weight: 600;
        letter-spacing: -0.02em;
        background: linear-gradient(
            90deg,
            rgba(255, 255, 255, 0.7),
            #fff 50%,
            rgba(255, 255, 255, 0.7)
        );
        background-size: 200% 100%;
        -webkit-background-clip: text;
        -webkit-text-fill-color: transparent;
        background-clip: text;
        animation: shimmer 2s ease-in-out infinite;
    }

    @keyframes shimmer {
        0% {
            background-position: 100% 50%;
        }
        50% {
            background-position: 0% 50%;
        }
        100% {
            background-position: 100% 50%;
        }
    }

    .loading-sub {
        font-size: 11px;
        font-weight: 500;
        letter-spacing: 0.1em;
        color: rgba(255, 255, 255, 0.25);
        text-transform: uppercase;
    }

    .loading-bar-wrap {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 8px;
        width: 220px;
        margin-top: 8px;
    }

    .loading-bar-track {
        width: 100%;
        height: 2px;
        background: rgba(255, 255, 255, 0.1);
        border-radius: 2px;
        overflow: hidden;
    }

    .loading-bar-fill {
        height: 100%;
        border-radius: 2px;
        background: linear-gradient(
            90deg,
            rgba(255, 255, 255, 0.4),
            rgba(255, 255, 255, 0.9)
        );
        transition: width 0.03s linear;
        box-shadow: 0 0 8px rgba(255, 255, 255, 0.4);
    }

    .loading-pct {
        font-size: 11px;
        font-weight: 500;
        letter-spacing: 0.06em;
        color: rgba(255, 255, 255, 0.25);
        font-variant-numeric: tabular-nums;
    }

    /* ── MENU ── */
    .menu-screen {
        position: absolute;
        inset: 0;
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 20;
        background: rgba(0, 0, 0, 0.7);
        backdrop-filter: blur(12px);
    }

    .menu-window {
        width: min(380px, 90vw);
        background: rgba(0, 0, 0, 0.5);
        border: 1px solid rgba(255, 255, 255, 0.06);
        border-radius: 9px;
        backdrop-filter: blur(20px);
        overflow: hidden;
        animation: winIn 0.35s cubic-bezier(0.16, 1, 0.3, 1);
        box-shadow: 0 40px 80px rgba(0, 0, 0, 0.5);
    }

    @keyframes winIn {
        from {
            transform: scale(0.92) translateY(16px);
            opacity: 0;
        }
        to {
            transform: none;
            opacity: 1;
        }
    }

    .menu-titlebar {
        display: flex;
        align-items: center;
        height: 28px;
        background: rgba(32, 32, 32, 0.85);
        backdrop-filter: blur(15px);
        border-radius: 9px 9px 0 0;
    }

    .menu-tl {
        display: flex;
        gap: 8px;
        padding: 0 12px;
        align-items: center;
    }

    .mtl-dot {
        width: 12px;
        height: 12px;
        border-radius: 50%;
        border: 0.5px solid #666;
        display: block;
    }

    .mtl-red {
        background: #ff5f57;
    }
    .mtl-yellow {
        background: #febc2e;
    }
    .mtl-green {
        background: #28c840;
    }

    .menu-titlebar-text {
        flex: 1;
        text-align: center;
        font-family:
            "SF Pro Display",
            -apple-system,
            BlinkMacSystemFont,
            sans-serif;
        font-size: 13px;
        font-weight: 500;
        color: rgba(255, 255, 255, 0.5);
        margin-right: 52px;
    }

    .menu-tl-spacer {
        width: 0;
    }

    .menu-titlebar-line {
        width: 100%;
        height: 1px;
        background: #2a2a2a;
    }

    .menu-body {
        padding: 20px;
        display: flex;
        flex-direction: column;
        gap: 14px;
    }

    .menu-heading {
        font-size: 13px;
        font-weight: 600;
        color: rgba(255, 255, 255, 0.4);
        letter-spacing: 0.04em;
        margin: 0;
    }

    .mode-select {
        display: flex;
        gap: 8px;
    }

    .mode-btn {
        flex: 1;
        display: flex;
        align-items: center;
        gap: 10px;
        padding: 12px 14px;
        background: rgba(255, 255, 255, 0.03);
        border: 1px solid rgba(255, 255, 255, 0.06);
        border-radius: 8px;
        color: rgba(255, 255, 255, 0.5);
        cursor: pointer;
        transition: all 0.2s;
        text-align: left;
    }

    .mode-btn:hover {
        background: rgba(255, 255, 255, 0.06);
        border-color: rgba(255, 255, 255, 0.1);
    }

    .mode-btn.active {
        background: rgba(99, 102, 241, 0.12);
        border-color: rgba(99, 102, 241, 0.35);
        color: #fff;
    }

    .mode-info {
        display: flex;
        flex-direction: column;
        gap: 1px;
    }

    .mode-name {
        font-size: 13px;
        font-weight: 600;
    }

    .mode-desc {
        font-size: 10px;
        opacity: 0.6;
    }

    .setting-row {
        display: flex;
        justify-content: space-between;
        align-items: center;
    }

    .setting-label {
        font-size: 12px;
        color: rgba(255, 255, 255, 0.4);
    }

    .setting-val {
        font-size: 12px;
        font-weight: 700;
        color: #6366f1;
        font-family: "SF Mono", monospace;
    }

    .sens-slider {
        width: 100%;
        height: 3px;
        -webkit-appearance: none;
        appearance: none;
        background: rgba(255, 255, 255, 0.08);
        border-radius: 2px;
        outline: none;
        margin-top: -6px;
    }

    .sens-slider::-webkit-slider-thumb {
        -webkit-appearance: none;
        width: 14px;
        height: 14px;
        background: #6366f1;
        border-radius: 50%;
        cursor: pointer;
        box-shadow: 0 0 8px rgba(99, 102, 241, 0.4);
    }

    .sens-slider::-moz-range-thumb {
        width: 14px;
        height: 14px;
        background: #6366f1;
        border-radius: 50%;
        cursor: pointer;
        border: none;
    }

    .play-btn {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 8px;
        width: 100%;
        padding: 13px;
        background: rgba(255, 255, 255, 0.06);
        border: 1px solid rgba(255, 255, 255, 0.08);
        border-radius: 8px;
        color: #fff;
        font-size: 14px;
        font-weight: 700;
        letter-spacing: 0.08em;
        cursor: pointer;
        transition: all 0.2s;
    }

    .play-btn:hover {
        background: rgba(255, 255, 255, 0.1);
        border-color: rgba(255, 255, 255, 0.15);
    }

    .play-btn:active {
        transform: scale(0.98);
    }

    .menu-hints {
        display: flex;
        justify-content: center;
        gap: 16px;
        color: rgba(255, 255, 255, 0.25);
        font-size: 11px;
    }

    .kbd {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        padding: 1px 6px;
        background: rgba(255, 255, 255, 0.06);
        border: 1px solid rgba(255, 255, 255, 0.08);
        border-radius: 4px;
        font-size: 10px;
        font-family: "SF Mono", monospace;
        font-weight: 600;
        color: rgba(255, 255, 255, 0.4);
        margin-right: 4px;
    }

    /* ── HUD ── */
    .hud {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        pointer-events: none;
        z-index: 10;
    }

    .hud-top {
        display: flex;
        align-items: flex-start;
        justify-content: space-between;
        padding: 16px 24px;
    }

    .hud-stat {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 2px;
    }

    .hud-label {
        font-size: 9px;
        font-weight: 700;
        letter-spacing: 0.15em;
        color: rgba(255, 255, 255, 0.35);
    }

    .hud-value {
        font-size: 24px;
        font-weight: 800;
        color: #fff;
        font-family: "SF Mono", monospace;
        text-shadow: 0 0 16px rgba(99, 102, 241, 0.4);
    }

    .score-val {
        color: #fbbf24;
        text-shadow: 0 0 16px rgba(251, 191, 36, 0.4);
    }

    .hud-timer {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 1px;
        padding: 10px 20px;
        background: rgba(0, 0, 0, 0.5);
        border-radius: 14px;
        border: 1px solid rgba(255, 255, 255, 0.06);
        backdrop-filter: blur(8px);
    }

    .hud-timer.urgent {
        border-color: rgba(244, 63, 94, 0.4);
        background: rgba(244, 63, 94, 0.12);
        animation: urgentPulse 0.5s ease infinite alternate;
    }

    @keyframes urgentPulse {
        to {
            box-shadow: 0 0 18px rgba(244, 63, 94, 0.25);
        }
    }

    .timer-big {
        font-size: 32px;
        font-weight: 900;
        color: #fff;
        font-family: "SF Mono", monospace;
        line-height: 1;
    }

    .timer-label {
        font-size: 9px;
        text-transform: uppercase;
        letter-spacing: 0.2em;
        color: rgba(255, 255, 255, 0.35);
    }

    .combo-pop {
        position: absolute;
        top: 80px;
        left: 50%;
        transform: translateX(-50%);
        display: flex;
        align-items: baseline;
        gap: 5px;
        animation: comboPop 0.2s cubic-bezier(0.16, 1, 0.3, 1);
    }

    @keyframes comboPop {
        from {
            transform: translateX(-50%) scale(1.4);
            opacity: 0.5;
        }
        to {
            transform: translateX(-50%) scale(1);
            opacity: 1;
        }
    }

    .combo-num {
        font-size: 22px;
        font-weight: 900;
        color: #fbbf24;
        font-family: "SF Mono", monospace;
        text-shadow: 0 0 12px rgba(251, 191, 36, 0.5);
    }

    .combo-word {
        font-size: 11px;
        font-weight: 800;
        color: rgba(251, 191, 36, 0.6);
        letter-spacing: 0.2em;
    }

    /* Crosshair */
    .crosshair {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        width: 22px;
        height: 22px;
        z-index: 15;
        pointer-events: none;
    }

    .ch-l {
        position: absolute;
        background: rgba(255, 255, 255, 0.85);
        border-radius: 1px;
    }

    .ch-t {
        width: 2px;
        height: 7px;
        top: 0;
        left: 50%;
        transform: translateX(-50%);
    }
    .ch-b {
        width: 2px;
        height: 7px;
        bottom: 0;
        left: 50%;
        transform: translateX(-50%);
    }
    .ch-le {
        width: 7px;
        height: 2px;
        left: 0;
        top: 50%;
        transform: translateY(-50%);
    }
    .ch-r {
        width: 7px;
        height: 2px;
        right: 0;
        top: 50%;
        transform: translateY(-50%);
    }
    .ch-dot {
        position: absolute;
        width: 3px;
        height: 3px;
        background: #fff;
        border-radius: 50%;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
    }

    .crosshair.hit .ch-l,
    .crosshair.hit .ch-dot {
        background: #4ade80;
        box-shadow: 0 0 6px rgba(74, 222, 128, 0.7);
    }
    .crosshair.miss .ch-l,
    .crosshair.miss .ch-dot {
        background: #f43f5e;
    }

    .hit-overlay {
        position: absolute;
        inset: 0;
        background: radial-gradient(
            ellipse at center,
            rgba(74, 222, 128, 0.06),
            transparent 70%
        );
        pointer-events: none;
        z-index: 5;
        animation: flashFade 0.15s ease-out forwards;
    }

    @keyframes flashFade {
        from {
            opacity: 1;
        }
        to {
            opacity: 0;
        }
    }

    .float-txt {
        position: absolute;
        top: 45%;
        left: 50%;
        transform: translateX(-50%);
        display: flex;
        flex-direction: column;
        align-items: center;
        pointer-events: none;
        z-index: 15;
        animation: floatUp 0.8s ease-out forwards;
    }

    @keyframes floatUp {
        0% {
            opacity: 1;
            transform: translateX(-50%) translateY(0);
        }
        100% {
            opacity: 0;
            transform: translateX(-50%) translateY(-50px);
        }
    }

    .ft-m {
        font-size: 18px;
        font-weight: 900;
        color: #fbbf24;
        font-family: "SF Mono", monospace;
        text-shadow: 0 0 8px rgba(251, 191, 36, 0.5);
    }
    .ft-s {
        font-size: 10px;
        font-weight: 800;
        color: rgba(251, 191, 36, 0.6);
        letter-spacing: 0.2em;
    }

    .mode-tag {
        position: absolute;
        bottom: 16px;
        left: 16px;
        font-size: 10px;
        font-weight: 700;
        letter-spacing: 0.12em;
        color: rgba(255, 255, 255, 0.25);
        pointer-events: none;
        z-index: 10;
    }

    /* ── RESULTS ── */
    .results-screen {
        position: absolute;
        inset: 0;
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 20;
        background: rgba(0, 0, 0, 0.7);
        backdrop-filter: blur(12px);
    }

    .results-window {
        width: min(420px, 90vw);
        background: rgba(0, 0, 0, 0.5);
        border: 1px solid rgba(255, 255, 255, 0.06);
        border-radius: 9px;
        backdrop-filter: blur(20px);
        overflow: hidden;
        animation: winIn 0.35s cubic-bezier(0.16, 1, 0.3, 1);
        box-shadow: 0 40px 80px rgba(0, 0, 0, 0.5);
    }

    .results-titlebar {
        display: flex;
        align-items: center;
        height: 28px;
        background: rgba(32, 32, 32, 0.85);
        backdrop-filter: blur(15px);
        border-radius: 9px 9px 0 0;
    }

    .results-body {
        padding: 20px;
        display: flex;
        flex-direction: column;
        gap: 18px;
    }

    .results-grid {
        display: grid;
        grid-template-columns: repeat(3, 1fr);
        gap: 10px;
    }

    .r-item {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 3px;
        padding: 12px 6px;
        background: rgba(255, 255, 255, 0.02);
        border-radius: 8px;
        border: 1px solid rgba(255, 255, 255, 0.04);
    }

    .r-val {
        font-size: 20px;
        font-weight: 900;
        color: #fff;
        font-family: "SF Mono", monospace;
    }

    .r-item.big .r-val {
        font-size: 28px;
        color: #fbbf24;
        text-shadow: 0 0 12px rgba(251, 191, 36, 0.35);
    }

    .r-lbl {
        font-size: 8px;
        font-weight: 700;
        letter-spacing: 0.15em;
        color: rgba(255, 255, 255, 0.3);
    }

    .results-btns {
        display: flex;
        gap: 8px;
    }

    .rbtn {
        flex: 1;
        padding: 12px;
        background: rgba(255, 255, 255, 0.04);
        border: 1px solid rgba(255, 255, 255, 0.08);
        border-radius: 8px;
        color: rgba(255, 255, 255, 0.6);
        font-size: 11px;
        font-weight: 700;
        letter-spacing: 0.06em;
        cursor: pointer;
        transition: all 0.2s;
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 6px;
    }

    .rbtn:hover {
        background: rgba(255, 255, 255, 0.08);
        color: #fff;
    }

    .rbtn.primary {
        flex: 2;
        background: rgba(255, 255, 255, 0.06);
        border-color: rgba(255, 255, 255, 0.1);
        color: #fff;
    }

    .rbtn.primary:hover {
        background: rgba(255, 255, 255, 0.1);
    }

    /* ── NICKNAME INPUT ── */
    .nick-body {
        align-items: center;
        text-align: center;
    }

    .nick-icon {
        font-size: 42px;
    }

    .nick-title {
        font-size: 22px;
        font-weight: 800;
        letter-spacing: 0.05em;
        background: linear-gradient(135deg, #6366f1, #f43f5e);
        -webkit-background-clip: text;
        -webkit-text-fill-color: transparent;
        background-clip: text;
        margin: 0;
    }

    .nick-sub {
        font-size: 12px;
        color: rgba(255, 255, 255, 0.35);
        margin: 0;
    }

    .nick-input {
        width: 100%;
        padding: 12px 14px;
        background: rgba(255, 255, 255, 0.04);
        border: 1px solid rgba(255, 255, 255, 0.1);
        border-radius: 8px;
        color: #fff;
        font-size: 14px;
        font-weight: 600;
        font-family:
            "SF Pro Display",
            -apple-system,
            BlinkMacSystemFont,
            sans-serif;
        outline: none;
        text-align: center;
        transition: border-color 0.2s;
    }

    .nick-input::placeholder {
        color: rgba(255, 255, 255, 0.2);
    }

    .nick-input:focus {
        border-color: rgba(99, 102, 241, 0.5);
        box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
    }

    .play-btn:disabled {
        opacity: 0.35;
        cursor: not-allowed;
    }

    /* ── TWO-COLUMN MENU ── */
    .menu-window-wide {
        width: min(680px, 92vw);
    }

    .menu-columns {
        display: flex;
        gap: 0;
    }

    .menu-columns .menu-body {
        flex: 1;
        border-right: 1px solid rgba(255, 255, 255, 0.05);
    }

    /* ── LEADERBOARD ── */
    .lb-panel {
        flex: 1;
        padding: 20px;
        display: flex;
        flex-direction: column;
        gap: 10px;
    }

    .lb-empty {
        font-size: 12px;
        color: rgba(255, 255, 255, 0.2);
        text-align: center;
        padding: 24px 0;
    }

    .lb-table {
        display: flex;
        flex-direction: column;
        gap: 3px;
    }

    .lb-header {
        display: flex;
        align-items: center;
        padding: 4px 8px;
        font-size: 9px;
        font-weight: 700;
        letter-spacing: 0.12em;
        color: rgba(255, 255, 255, 0.25);
        text-transform: uppercase;
    }

    .lb-row {
        display: flex;
        align-items: center;
        padding: 7px 8px;
        background: rgba(255, 255, 255, 0.02);
        border-radius: 6px;
        font-size: 12px;
        color: rgba(255, 255, 255, 0.6);
        transition: background 0.15s;
    }

    .lb-row:hover {
        background: rgba(255, 255, 255, 0.05);
    }
    .lb-row.lb-me {
        background: rgba(99, 102, 241, 0.1);
        border: 1px solid rgba(99, 102, 241, 0.2);
        color: #fff;
    }
    .lb-row.lb-gold .lb-rank,
    .lb-row.lb-gold .lb-name {
        color: #fbbf24;
    }
    .lb-row.lb-silver .lb-rank,
    .lb-row.lb-silver .lb-name {
        color: #cbd5e1;
    }
    .lb-row.lb-bronze .lb-rank,
    .lb-row.lb-bronze .lb-name {
        color: #d97706;
    }

    .lb-rank {
        width: 28px;
        flex-shrink: 0;
        font-weight: 700;
    }
    .lb-name {
        flex: 1;
        font-weight: 600;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
    .lb-score {
        width: 52px;
        text-align: right;
        font-weight: 800;
        font-family: "SF Mono", monospace;
        color: #fbbf24;
    }
    .lb-acc {
        width: 38px;
        text-align: right;
        font-weight: 600;
        font-family: "SF Mono", monospace;
        opacity: 0.6;
    }

    /* ── RESULTS LEADERBOARD ── */
    .results-lb {
        display: flex;
        flex-direction: column;
        gap: 8px;
    }

    .results-lb-title {
        font-size: 12px;
        font-weight: 700;
        letter-spacing: 0.04em;
        color: rgba(255, 255, 255, 0.4);
        margin: 0;
    }

    .lb-compact .lb-row {
        padding: 5px 6px;
        font-size: 11px;
    }

    .lb-compact .lb-score {
        width: 46px;
    }
</style>
