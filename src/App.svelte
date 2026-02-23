<script>
  import { onMount } from "svelte";
  import { spring } from "svelte/motion";
  import LoadingScreen from "./LoadingScreen.svelte";

  const base = import.meta.env.BASE_URL;

  let show = true;
  let showPlayer = true;
  let isDarkTheme = false;

  function toggleTheme() {
    isDarkTheme = !isDarkTheme;

    // Sync music with theme
    if (audio) {
      if (isDarkTheme) {
        currentTrackIndex = 2; // track3.mp3
      } else {
        currentTrackIndex = 0; // track.mp3
      }
      loadTrack(currentTrackIndex, true);
    }
  }

  function togglePlayer() {
    showPlayer = !showPlayer;
  }

  function openBioApp() {
    if (!show) {
      show = true;
      typedText = "";
      showCursor = true;
      setTimeout(typeWriter, 200);
    } else {
      handleClose();
    }
  }
  let siteEntered = false;
  let minimized = false;
  let closed = false;
  let windowEl;
  let isDragging = false;
  let dragOffset = { x: 0, y: 0 };
  let position = { x: 0, y: 0 };
  let hasMoved = false;

  // Mobile detection
  let isMobile = false;

  // Typewriter effect
  let typedText = "";
  const fullText = "реверс инжиниринг разработчик и музыкант";
  let showCursor = true;

  // Particles
  let particles = [];
  let particleId = 0;

  const links = [
    {
      id: 1,
      title: "my tg channel",
      icon: "telegram",
      url: "https://t.me/haosyarchives",
      tooltip: "Открыть Telegram",
      tooltipPos: "top",
    },
    {
      id: 2,
      title: "spotify profile",
      icon: "spotify",
      url: "https://open.spotify.com/user/8r02yukempwltp3jj1jajx5dc?si=8ccd209bdda049ea",
      tooltip: "Открыть Spotify",
      tooltipPos: "top",
    },
    {
      id: 3,
      title: "discord",
      icon: "discord",
      url: "https://discord.gg/ffGYWDWczc",
      tooltip: "Открыть Discord",
      tooltipPos: "top",
    },
    {
      id: 4,
      title: "soundcloud",
      icon: "soundcloud",
      url: "https://soundcloud.com/haos1y",
      tooltip: "Открыть Soundcloud",
      tooltipPos: "top",
    },
  ];

  // Avatar tilt when dragging
  let avatarTilt = { x: 0, y: 0 };

  // Avatar glitch easter egg
  let avatarClicks = 0;
  let isGlitching = false;

  // Notification badge
  let showBadge = false;
  let badgeCount = 1;

  // Easter egg terminal crash
  let typedKeys = "";
  let showTerminal = false;
  let terminalLines = [];
  let terminalInput = "";
  let terminalError = false;

  function startTerminalCrash() {
    showTerminal = true;
    terminalLines = ["Initializing root access protocol..."];
    terminalInput = "";
    terminalError = false;

    setTimeout(
      () =>
        pushLine("Connecting to local secure server [127.0.0.1]... SUCCESS"),
      800,
    );
    setTimeout(() => pushLine("Handshake established."), 1400);
    setTimeout(() => pushLine("enter login and password"), 2000);

    // Animate typing login
    setTimeout(() => typeInput("login: admin"), 2600);
    setTimeout(() => pushLine("login: admin"), 3800); // 2600 + 1200ms type time

    // Animate typing password
    setTimeout(() => typeInput("password: *****"), 4200);
    setTimeout(() => pushLine("password: *****"), 5400);
    setTimeout(() => pushLine("Verifying credentials..."), 5800);

    // Crash
    setTimeout(() => {
      terminalError = true;
      pushLine("[!!!] ACCESS DENIED");
      pushLine("[!!!] FATAL SECURITY EXCEPTION");
      pushLine("[!!!] INITIATING SYSTEM LOCKDOWN");
    }, 7000);
  }

  function pushLine(text) {
    terminalLines = [...terminalLines, text];
    terminalInput = ""; // Clear input when a line is finished
  }

  function typeInput(text) {
    let i = 0;
    terminalInput = "";
    const interval = setInterval(() => {
      if (i < text.length) {
        terminalInput += text[i];
        i++;
      } else {
        clearInterval(interval);
      }
    }, 80); // 80ms per character
  }

  // Audio player physics (inertia)
  let isDraggingPlayer = false;
  let playerDragOffset = { x: 0, y: 0 };
  let playerPos = spring(
    { x: 0, y: 0 },
    {
      stiffness: 0.08,
      damping: 0.35, // Smooth inertia and slight overshoot
    },
  );

  function startPlayerDrag(e) {
    if (e.target.closest("button, .audio-progress-container, .volume-slider"))
      return;
    isDraggingPlayer = true;
    playerDragOffset.x = e.clientX - $playerPos.x;
    playerDragOffset.y = e.clientY - $playerPos.y;
  }

  // Audio player
  let audio;
  let isPlaying = false;
  let audioProgress = 0;
  let audioDuration = 0;
  let audioCurrentTime = 0;
  let audioVolume = 0.3;
  let isDraggingVolume = false;
  let isDraggingProgress = false;

  // Audio Visualizer
  let audioContext;
  let analyser;
  let source;
  let dataArray;
  let canvasEl;
  let canvasCtx;
  let visualizerInitialized = false;

  function initVisualizer() {
    if (visualizerInitialized || !audio) return;

    audioContext = new (window.AudioContext || window.webkitAudioContext)();
    analyser = audioContext.createAnalyser();

    source = audioContext.createMediaElementSource(audio);
    source.connect(analyser);
    analyser.connect(audioContext.destination);

    analyser.fftSize = 256; // 64 bins for richer visualizer
    const bufferLength = analyser.frequencyBinCount;
    dataArray = new Uint8Array(bufferLength);

    if (canvasEl) {
      canvasEl.width = window.innerWidth;
      canvasEl.height = window.innerHeight;
      canvasCtx = canvasEl.getContext("2d");
      drawVisualizer();
    }

    visualizerInitialized = true;
  }

  function drawVisualizer() {
    requestAnimationFrame(drawVisualizer);
    if (!canvasCtx || !analyser) return;

    analyser.getByteFrequencyData(dataArray);

    canvasCtx.clearRect(0, 0, canvasEl.width, canvasEl.height);

    const barWidth = (canvasEl.width / dataArray.length) * 2.5;
    let barHeight;
    let x = 0;

    for (let i = 0; i < dataArray.length; i++) {
      barHeight = dataArray[i] * 6.5; // Scale height up significantly for a taller visualizer

      // Create gradient for bars (syncs with light/dark theme style)
      const gradient = canvasCtx.createLinearGradient(
        0,
        canvasEl.height,
        0,
        canvasEl.height - barHeight,
      );
      if (isDarkTheme) {
        gradient.addColorStop(0, "rgba(255, 255, 255, 0.1)");
        gradient.addColorStop(1, "rgba(255, 255, 255, 0.7)");
      } else {
        gradient.addColorStop(0, "rgba(0, 0, 0, 0.1)");
        gradient.addColorStop(1, "rgba(0, 0, 0, 0.6)");
      }

      canvasCtx.fillStyle = gradient;
      canvasCtx.fillRect(
        x,
        canvasEl.height - barHeight / 2,
        barWidth,
        barHeight / 2,
      );

      x += barWidth + 2;
    }
  }

  // Playlist
  let currentTrackIndex = 0;
  const tracks = [
    {
      name: "faustian bargain",
      artist: "королевский XVII",
      src: base + "track.mp3",
      cover: base + "track.webp",
    },
    {
      name: "болеть тобой x mylancore",
      artist: "qwzbtw0",
      src: base + "track2.mp3",
      cover: base + "track2.webp",
    },
    {
      name: "Romantica.com",
      artist: "Full Smena, ego10sei",
      src: base + "track3.mp3",
      cover: base + "track3.webp",
    },
  ];

  $: trackName = tracks[currentTrackIndex].name;
  $: artistName = tracks[currentTrackIndex].artist;
  $: trackCover = tracks[currentTrackIndex].cover;

  // Notifications (Dynamic Island style)
  let notifications = [];
  let notificationId = 0;

  function showNotification(title, message, icon = "info") {
    const id = notificationId++;
    notifications = [
      ...notifications,
      { id, title, message, icon, visible: true },
    ];

    // Auto dismiss after 4 seconds
    setTimeout(() => {
      dismissNotification(id);
    }, 4000);
  }

  function dismissNotification(id) {
    notifications = notifications.map((n) =>
      n.id === id ? { ...n, visible: false } : n,
    );
    setTimeout(() => {
      notifications = notifications.filter((n) => n.id !== id);
    }, 300);
  }

  // Depth shadow based on mouse position
  let shadowX = 0;
  let shadowY = 0;

  // Custom cursor shake to find
  let cursorEl;
  let cursorX = -100;
  let cursorY = -100;
  let cursorScale = 1;
  let lastMouseX = 0;
  let speedHistory = [];
  let isPointer = false;
  let cursorReady = false;

  // Context menu
  let contextMenuOpen = false;
  let contextMenuX = 0;
  let contextMenuY = 0;

  const contextMenuItems = [
    {
      icon: "copy",
      label: "Копировать ссылку",
      action: () =>
        navigator.clipboard.writeText("https://haosyy.github.io/haos1y.site/"),
    },
    {
      icon: "share",
      label: "Поделиться",
      action: () =>
        navigator.share?.({
          url: "https://haosyy.github.io/haos1y.site/",
          title: "haos1y",
        }),
    },
    { type: "separator" },
    {
      icon: "telegram",
      label: "Telegram",
      action: () => window.open("https://t.me/haosyarchives", "_blank"),
    },
    {
      icon: "spotify",
      label: "Spotify",
      action: () =>
        window.open(
          "https://open.spotify.com/user/8r02yukempwltp3jj1jajx5dc?si=36390fa567724146",
          "_blank",
        ),
    },
    { type: "separator" },
    {
      icon: "search",
      label: "Поиск",
      shortcut: "⌘K",
      action: () => {
        spotlightOpen = true;
        setTimeout(() => searchInputEl?.focus(), 100);
      },
    },
  ];

  // Spotlight search
  let spotlightOpen = false;
  let searchQuery = "";
  let searchInputEl;

  const spotlightItems = [
    {
      icon: "person",
      title: "Обо мне",
      desc: "haos1y — реверс инжиниринг разработчик и музыкант",
      action: () => {},
    },
    {
      icon: "telegram",
      title: "Telegram",
      desc: "Открыть канал в Telegram",
      action: () => window.open("https://t.me/haosyarchives", "_blank"),
    },
    {
      icon: "spotify",
      title: "Spotify",
      desc: "Открыть профиль Spotify",
      action: () =>
        window.open(
          "https://open.spotify.com/user/8r02yukempwltp3jj1jajx5dc?si=36390fa567724146",
          "_blank",
        ),
    },
    {
      icon: "discord",
      title: "Discord",
      desc: "Присоединиться к серверу Discord",
      action: () => window.open("https://discord.gg/ffGYWDWczc", "_blank"),
    },
    {
      icon: "soundcloud",
      title: "Soundcloud",
      desc: "Открыть профиль Soundcloud",
      action: () => window.open("https://soundcloud.com/haos1y", "_blank"),
    },
    {
      icon: "star",
      title: "Пасхалка",
      desc: "Кликни 5 раз на аватар",
      action: () => {
        spotlightOpen = false;
        triggerGlitch();
      },
    },
    {
      icon: "keyboard",
      title: "Горячие клавиши",
      desc: "⌘K — поиск, Esc — закрыть",
      action: () => {},
    },
    {
      icon: "cursor",
      title: "Shake to find",
      desc: "Потряси мышкой чтобы найти курсор",
      action: () => {},
    },
    {
      icon: "github",
      title: "Made by",
      desc: "haos1y",
      action: () => window.open("https://github.com/haosyy", "_blank"),
    },
  ];

  $: filteredItems = searchQuery
    ? spotlightItems.filter(
        (item) =>
          item.title.toLowerCase().includes(searchQuery.toLowerCase()) ||
          item.desc.toLowerCase().includes(searchQuery.toLowerCase()),
      )
    : spotlightItems;

  // Typewriter effect
  function typeWriter() {
    let i = 0;
    const interval = setInterval(() => {
      if (i < fullText.length) {
        typedText = fullText.slice(0, i + 1);
        i++;
      } else {
        clearInterval(interval);
        setTimeout(() => (showCursor = false), 1000);
      }
    }, 80);
  }

  // Particles effect
  function createParticles(e) {
    const rect = e.currentTarget.getBoundingClientRect();
    const x = rect.left + rect.width / 2;
    const y = rect.top + rect.height / 2;

    for (let i = 0; i < 8; i++) {
      const angle = (i / 8) * Math.PI * 2;
      const velocity = 2 + Math.random() * 2;
      particles = [
        ...particles,
        {
          id: particleId++,
          x,
          y,
          vx: Math.cos(angle) * velocity,
          vy: Math.sin(angle) * velocity,
          life: 1,
        },
      ];
    }

    requestAnimationFrame(animateParticles);
  }

  function animateParticles() {
    particles = particles
      .map((p) => ({
        ...p,
        x: p.x + p.vx * 3,
        y: p.y + p.vy * 3,
        life: p.life - 0.03,
      }))
      .filter((p) => p.life > 0);

    if (particles.length > 0) {
      requestAnimationFrame(animateParticles);
    }
  }

  // Avatar glitch
  function handleAvatarClick() {
    avatarClicks++;
    if (avatarClicks >= 5) {
      triggerGlitch();
      avatarClicks = 0;
    }
  }

  function triggerGlitch() {
    isGlitching = true;
    setTimeout(() => {
      isGlitching = false;
    }, 1500);
  }

  // Custom cursor movement with shake to find
  function handleMouseMove(e) {
    cursorX = e.clientX;
    cursorY = e.clientY;

    // First move - initialize
    if (!cursorReady) {
      cursorReady = true;
      lastMouseX = e.clientX;
      return;
    }

    // Check if hovering over clickable element
    const target = e.target;
    isPointer =
      target.closest(
        'a, button, [role="button"], .avatar-container, .spotlight-item, .context-menu-item, .keyboard-hint, .notification',
      ) !== null;

    // Calculate speed for shake detection
    const deltaX = Math.abs(e.clientX - lastMouseX);
    speedHistory.push(deltaX);
    if (speedHistory.length > 8) speedHistory.shift();

    const avgSpeed =
      speedHistory.reduce((a, b) => a + b, 0) / speedHistory.length;

    // Scale cursor based on shake speed (macOS shake to find)
    if (avgSpeed > 30) {
      cursorScale = Math.min(5, 1 + avgSpeed / 30);
    } else {
      cursorScale = Math.max(1, cursorScale - 0.1);
    }

    lastMouseX = e.clientX;

    // Depth shadow effect
    const centerX = window.innerWidth / 2;
    const centerY = window.innerHeight / 2;
    shadowX = (e.clientX - centerX) / 30;
    shadowY = (e.clientY - centerY) / 30;

    // Also handle drag
    drag(e);
  }

  // Keyboard shortcuts and easter egg tracker
  function handleKeydown(e) {
    // Escape to close spotlight or context menu
    if (e.key === "Escape") {
      if (spotlightOpen) closeSpotlight();
      if (contextMenuOpen) closeContextMenu();
      if (showTerminal) {
        showTerminal = false;
        terminalLines = [];
        terminalError = false;
      }
    }

    // Ignore tracker if in spotlight input
    if (spotlightOpen || showTerminal) return;

    // Cmd/Ctrl + K for spotlight
    if ((e.metaKey || e.ctrlKey) && e.key === "k") {
      e.preventDefault();
      spotlightOpen = !spotlightOpen;
      searchQuery = "";
      if (spotlightOpen) {
        setTimeout(() => searchInputEl?.focus(), 100);
      }
      return;
    }

    // Easter egg tracking
    if (e.key.length === 1) {
      // Only track single characters
      typedKeys += e.key.toLowerCase();
      if (typedKeys.length > 20) typedKeys = typedKeys.slice(-20);

      if (typedKeys.endsWith("admin")) {
        typedKeys = "";
        startTerminalCrash();
      }
    }
  }

  function handleSpotlightItemClick(item) {
    item.action();
    if (item.action.toString() !== "() => {}") {
      spotlightOpen = false;
      searchQuery = "";
    }
  }

  function closeSpotlight() {
    const overlay = document.querySelector(".spotlight-overlay");
    const spotlight = document.querySelector(".spotlight");
    if (overlay) overlay.classList.add("closing");
    if (spotlight) spotlight.classList.add("closing");
    setTimeout(() => {
      spotlightOpen = false;
      searchQuery = "";
    }, 200);
  }

  // Context menu
  function handleContextMenu(e) {
    e.preventDefault();
    contextMenuX = e.clientX;
    contextMenuY = e.clientY;
    contextMenuOpen = true;
  }

  function closeContextMenu() {
    contextMenuOpen = false;
  }

  function handleContextMenuItemClick(item) {
    item.action();
    contextMenuOpen = false;
  }

  // Double click on title bar
  function handleTitleBarDoubleClick(e) {
    if (e.target.closest(".traffic-lights")) return;
    handleMinimize();
  }

  onMount(() => {
    // Detect mobile / touch device
    isMobile =
      "ontouchstart" in window ||
      navigator.maxTouchPoints > 0 ||
      window.innerWidth < 768;
    if (isMobile) {
      document.body.classList.add("is-mobile");
    }

    requestAnimationFrame(() => {
      show = true;
      setTimeout(typeWriter, 800);
    });

    // Welcome notification
    setTimeout(() => {
      showNotification(
        "Добро пожаловать",
        isMobile ? "Добро пожаловать!" : "shortcut",
        "wave",
      );
    }, 2000);

    // Telegram notification + badge
    setTimeout(() => {
      showBadge = true;
      showNotification("Telegram", "Новый пост в канале", "telegram");
    }, 6000);

    // Init audio
    audio = new Audio(tracks[currentTrackIndex].src);
    // Don't loop a single audio unconditionally anymore if we have a playlist, handle it in timeupdate
    audio.volume = 0; // Start at 0 for fade in

    audio.addEventListener("timeupdate", () => {
      if (!isDraggingProgress) {
        audioProgress = (audio.currentTime / audio.duration) * 100 || 0;
        audioCurrentTime = audio.currentTime;
      }
    });

    audio.addEventListener("ended", () => {
      nextTrack();
    });

    audio.addEventListener("loadedmetadata", () => {
      audioDuration = audio.duration;
    });
  });

  // Start audio with fade in when user enters site
  function handleSiteEnter() {
    siteEntered = true;
    if (!audio) {
      console.error("Audio not initialized");
      return;
    }

    if (audioContext && audioContext.state === "suspended") {
      audioContext.resume();
    }

    audio.volume = 0;
    audio
      .play()
      .then(() => {
        isPlaying = true;
        initVisualizer(); // Initialize Web Audio API on first interaction

        // Fade in volume over 2 seconds
        let vol = 0;
        const fadeIn = setInterval(() => {
          vol += 0.015;
          if (vol >= audioVolume) {
            audio.volume = audioVolume;
            clearInterval(fadeIn);
          } else {
            audio.volume = vol;
          }
        }, 50);
      })
      .catch((err) => {
        console.error("Audio play failed:", err);
        isPlaying = false;
      });
  }

  function toggleAudio() {
    if (isPlaying) {
      audio.pause();
    } else {
      audio.play();
    }
    isPlaying = !isPlaying;
  }

  function loadTrack(index, autoPlay = true) {
    const wasPlaying = isPlaying;
    audio.src = tracks[index].src;
    audio.load();
    audioProgress = 0;
    audioCurrentTime = 0;

    if (autoPlay && wasPlaying) {
      audio.play().catch((err) => console.error("Audio play error:", err));
      isPlaying = true;
    }
  }

  function nextTrack() {
    currentTrackIndex = (currentTrackIndex + 1) % tracks.length;
    loadTrack(currentTrackIndex, isPlaying);
  }

  function prevTrack() {
    if (audio.currentTime > 3) {
      audio.currentTime = 0; // If playing for >3 seconds, just restart the track
    } else {
      currentTrackIndex =
        (currentTrackIndex - 1 + tracks.length) % tracks.length;
      loadTrack(currentTrackIndex, isPlaying);
    }
  }

  function formatTime(seconds) {
    if (!seconds || isNaN(seconds)) return "0:00";
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins}:${secs.toString().padStart(2, "0")}`;
  }

  function seekAudio(e) {
    const rect = e.currentTarget.getBoundingClientRect();
    const percent = Math.max(
      0,
      Math.min(1, (e.clientX - rect.left) / rect.width),
    );
    audio.currentTime = percent * audio.duration;
    audioProgress = percent * 100;
  }

  function startProgressDrag(e) {
    isDraggingProgress = true;
    seekAudio(e);
  }

  function setVolume(e) {
    const rect = e.currentTarget.getBoundingClientRect();
    const percent = Math.max(
      0,
      Math.min(1, (e.clientX - rect.left) / rect.width),
    );
    audioVolume = percent;
    audio.volume = audioVolume;
  }

  function startVolumeDrag(e) {
    isDraggingVolume = true;
    setVolume(e);
  }

  function handleGlobalMouseMove(e) {
    if (isDraggingPlayer) {
      playerPos.set({
        x: e.clientX - playerDragOffset.x,
        y: e.clientY - playerDragOffset.y,
      });
    }
    if (isDraggingVolume) {
      const slider = document.querySelector(".volume-slider");
      if (slider) {
        const rect = slider.getBoundingClientRect();
        const percent = Math.max(
          0,
          Math.min(1, (e.clientX - rect.left) / rect.width),
        );
        audioVolume = percent;
        audio.volume = audioVolume;
      }
    }
    if (isDraggingProgress) {
      const progress = document.querySelector(".audio-progress");
      if (progress) {
        const rect = progress.getBoundingClientRect();
        const percent = Math.max(
          0,
          Math.min(1, (e.clientX - rect.left) / rect.width),
        );
        audioProgress = percent * 100;
        audioCurrentTime = percent * audio.duration;
      }
    }
  }

  function handleGlobalMouseUp() {
    isDraggingPlayer = false;
    if (isDraggingProgress && audio) {
      audio.currentTime = (audioProgress / 100) * audio.duration;
    }
    isDraggingVolume = false;
    isDraggingProgress = false;
  }

  function handleClose() {
    closed = true;
    setTimeout(() => {
      closed = false;
      show = false;
      typedText = "";
      showCursor = true;
      setTimeout(() => {
        show = true;
        setTimeout(typeWriter, 500);
      }, 300);
    }, 500);
  }

  function handleMinimize() {
    minimized = true;
    setTimeout(() => {
      minimized = false;
    }, 600);
  }

  function handleZoom() {
    window.open("https://t.me/haosyarchives", "_blank");
  }

  function handleLinkClick(e) {
    // Hide badge when telegram clicked
    const link = e.currentTarget;
    if (link.href.includes("t.me")) {
      showBadge = false;
    }
  }

  // Drag functionality
  function startDrag(e) {
    if (isMobile) return; // Disable drag on mobile
    if (e.target.closest(".traffic-lights")) return;
    isDragging = true;
    const rect = windowEl.getBoundingClientRect();
    dragOffset.x = e.clientX - rect.left - rect.width / 2;
    dragOffset.y = e.clientY - rect.top - rect.height / 2;
    if (!hasMoved) {
      position.x = 0;
      position.y = 0;
    }
  }

  function drag(e) {
    if (!isDragging) return;
    hasMoved = true;
    const centerX = window.innerWidth / 2;
    const centerY = window.innerHeight / 2;

    let newX = e.clientX - centerX - dragOffset.x;
    let newY = e.clientY - centerY - dragOffset.y;

    // Avatar tilt based on movement
    const deltaX = newX - position.x;
    const deltaY = newY - position.y;
    avatarTilt = {
      x: Math.max(-15, Math.min(15, deltaX * 2)),
      y: Math.max(-15, Math.min(15, deltaY * 2)),
    };

    // Границы окна (окно не выходит за края)
    const windowWidth = 600;
    const windowHeight = 560;
    const minX = -centerX + windowWidth / 2;
    const maxX = centerX - windowWidth / 2;
    const minY = -centerY + windowHeight / 2;
    const maxY = centerY - windowHeight / 2;

    position.x = Math.max(minX, Math.min(maxX, newX));
    position.y = Math.max(minY, Math.min(maxY, newY));
  }

  function stopDrag() {
    isDragging = false;
    avatarTilt = { x: 0, y: 0 };
  }

  // Touch support for audio progress bar
  function handleProgressTouch(e) {
    e.preventDefault();
    const touch = e.touches[0];
    const progress =
      e.currentTarget.querySelector(".audio-progress") || e.currentTarget;
    const rect = progress.getBoundingClientRect();
    const percent = Math.max(
      0,
      Math.min(1, (touch.clientX - rect.left) / rect.width),
    );
    audio.currentTime = percent * audio.duration;
    audioProgress = percent * 100;
    audioCurrentTime = audio.currentTime;
  }

  // Touch support for volume slider
  function handleVolumeTouch(e) {
    e.preventDefault();
    const touch = e.touches[0];
    const slider = e.currentTarget;
    const rect = slider.getBoundingClientRect();
    const percent = Math.max(
      0,
      Math.min(1, (touch.clientX - rect.left) / rect.width),
    );
    audioVolume = percent;
    audio.volume = audioVolume;
  }
</script>

<svelte:window
  on:mousemove={(e) => {
    handleMouseMove(e);
    handleGlobalMouseMove(e);
  }}
  on:mouseup={() => {
    stopDrag();
    handleGlobalMouseUp();
  }}
  on:keydown={handleKeydown}
  on:click={closeContextMenu}
/>

<!-- Notifications -->
<div class="notifications">
  {#each notifications as notification (notification.id)}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
      class="notification"
      class:hiding={!notification.visible}
      on:click={() => dismissNotification(notification.id)}
    >
      <div class="notification-icon">
        {#if notification.icon === "wave"}
          <svg
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="1.5"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path d="M18.5 8c.5-1.5 0-3-1.5-3.5s-3 .5-3.5 2l-1 3" />
            <path d="M14.5 9.5c.5-1.5 0-3-1.5-3.5s-3 .5-3.5 2l-.5 2" />
            <path
              d="M10.5 11c.5-1.5 0-3-1.5-3.5s-3 .5-3.5 2l-1 4c-1 4 1 7 5 8s7-1 8-5l2-6c.5-1.5 0-3-1.5-3.5s-3 .5-3.5 2l-1 3"
            />
          </svg>
        {:else if notification.icon === "telegram"}
          <img src="{base}telegram.svg" alt="" width="24" height="24" />
        {:else if notification.icon === "check"}
          <svg
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="#34C759"
            stroke-width="2"
          >
            <path d="M20 6L9 17l-5-5" />
          </svg>
        {:else}
          <svg
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="1.5"
          >
            <circle cx="12" cy="12" r="10" />
            <path d="M12 16v-4M12 8h.01" />
          </svg>
        {/if}
      </div>
      <div class="notification-content">
        {#if notification.message === "shortcut"}
          <span class="notification-title">{notification.title}</span>
          <span class="notification-message">
            Нажми <span class="notif-key">⌘</span><span class="notif-key"
              >K</span
            > для поиска
          </span>
        {:else}
          <span class="notification-title">{notification.title}</span>
          <span class="notification-message">{notification.message}</span>
        {/if}
      </div>
    </div>
  {/each}
</div>

<!-- Custom cursor -->
<div
  class="custom-cursor"
  class:pointer={isPointer}
  class:ready={cursorReady}
  bind:this={cursorEl}
  style="
    left: {cursorX}px;
    top: {cursorY}px;
    transform: translate(-20%, -15%) scale({cursorScale});
  "
>
  {#if isPointer}
    <img src="{base}cursor-pointer.svg" alt="" draggable="false" />
  {:else}
    <img src="{base}cursor-default.svg" alt="" draggable="false" />
  {/if}
</div>

<LoadingScreen on:enter={handleSiteEnter} />

<!-- Spotlight Search -->
{#if spotlightOpen}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div class="spotlight-overlay" on:click={closeSpotlight}>
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="spotlight" on:click|stopPropagation>
      <div class="spotlight-input-wrapper">
        <svg
          class="spotlight-icon"
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <circle cx="11" cy="11" r="8" />
          <path d="m21 21-4.35-4.35" />
        </svg>
        <input
          type="text"
          class="spotlight-input"
          placeholder="Поиск..."
          bind:value={searchQuery}
          bind:this={searchInputEl}
        />
        <span class="spotlight-shortcut">esc</span>
      </div>
      <div class="spotlight-results">
        {#each filteredItems as item}
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <!-- svelte-ignore a11y-no-static-element-interactions -->
          <div
            class="spotlight-item"
            on:click={() => handleSpotlightItemClick(item)}
          >
            <div class="spotlight-item-icon">
              {#if item.icon === "person"}
                <svg
                  width="20"
                  height="20"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="1.5"
                >
                  <circle cx="12" cy="8" r="4" />
                  <path d="M4 20c0-4 4-6 8-6s8 2 8 6" />
                </svg>
              {:else if item.icon === "telegram"}
                <img src="{base}telegram.svg" alt="" width="20" height="20" />
              {:else if item.icon === "spotify"}
                <img src="{base}spotify.svg" alt="" width="20" height="20" />
              {:else if item.icon === "discord"}
                <img src="{base}discord.svg" alt="" width="20" height="20" />
              {:else if item.icon === "soundcloud"}
                <img src="{base}soundcloud.svg" alt="" width="20" height="20" />
              {:else if item.icon === "star"}
                <svg
                  width="20"
                  height="20"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="1.5"
                >
                  <path
                    d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"
                  />
                </svg>
              {:else if item.icon === "keyboard"}
                <svg
                  width="20"
                  height="20"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="1.5"
                >
                  <rect x="2" y="4" width="20" height="16" rx="2" />
                  <path
                    d="M6 8h.01M10 8h.01M14 8h.01M18 8h.01M6 12h.01M10 12h.01M14 12h.01M18 12h.01M8 16h8"
                  />
                </svg>
              {:else if item.icon === "cursor"}
                <svg
                  width="20"
                  height="20"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="1.5"
                >
                  <path d="M4 4l7.07 17 2.51-7.39L21 11.07 4 4z" />
                </svg>
              {:else if item.icon === "github"}
                <svg
                  width="20"
                  height="20"
                  viewBox="0 0 24 24"
                  fill="currentColor"
                >
                  <path
                    d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"
                  />
                </svg>
              {/if}
            </div>
            <div class="spotlight-item-content">
              <span class="spotlight-item-title">{item.title}</span>
              <span class="spotlight-item-desc">{item.desc}</span>
            </div>
          </div>
        {/each}
        {#if filteredItems.length === 0}
          <div class="spotlight-empty">Ничего не найдено</div>
        {/if}
      </div>
    </div>
  </div>
{/if}

<!-- Context Menu -->
{#if contextMenuOpen}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div
    class="context-menu"
    style="left: {contextMenuX}px; top: {contextMenuY}px;"
    on:click|stopPropagation
  >
    {#each contextMenuItems as item}
      {#if item.type === "separator"}
        <div class="context-menu-separator"></div>
      {:else}
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <!-- svelte-ignore a11y-no-static-element-interactions -->
        <div
          class="context-menu-item"
          on:click={() => handleContextMenuItemClick(item)}
        >
          <div class="context-menu-icon">
            {#if item.icon === "copy"}
              <svg
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="1.5"
              >
                <rect x="9" y="9" width="13" height="13" rx="2" />
                <path d="M5 15H4a2 2 0 01-2-2V4a2 2 0 012-2h9a2 2 0 012 2v1" />
              </svg>
            {:else if item.icon === "share"}
              <svg
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="1.5"
              >
                <path d="M4 12v8a2 2 0 002 2h12a2 2 0 002-2v-8" />
                <polyline points="16,6 12,2 8,6" />
                <line x1="12" y1="2" x2="12" y2="15" />
              </svg>
            {:else if item.icon === "telegram"}
              <img src="{base}telegram.svg" alt="" width="16" height="16" />
            {:else if item.icon === "spotify"}
              <img src="{base}spotify.svg" alt="" width="16" height="16" />
            {:else if item.icon === "search"}
              <svg
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="1.5"
              >
                <circle cx="11" cy="11" r="8" />
                <path d="m21 21-4.35-4.35" />
              </svg>
            {/if}
          </div>
          <span class="context-menu-label">{item.label}</span>
          {#if item.shortcut}
            <span class="context-menu-shortcut">{item.shortcut}</span>
          {/if}
        </div>
      {/if}
    {/each}
  </div>
{/if}

<!-- Keyboard hint -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
  class="keyboard-hint"
  class:hidden={spotlightOpen}
  on:click={() => {
    spotlightOpen = true;
    setTimeout(() => searchInputEl?.focus(), 100);
  }}
>
  <svg
    width="14"
    height="14"
    viewBox="0 0 24 24"
    fill="none"
    stroke="currentColor"
    stroke-width="2"
  >
    <circle cx="11" cy="11" r="8" />
    <path d="m21 21-4.35-4.35" />
  </svg>
  <span class="hint-text">поиск</span>
  <div class="hint-keys">
    <span class="key">⌘</span><span class="key">K</span>
  </div>
</div>

<!-- Audio player - Apple Music style -->
{#if showPlayer}
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div
    class="audio-player"
    class:playing={isPlaying}
    style="transform: translate({isMobile ? 0 : $playerPos.x}px, {isMobile
      ? 0
      : $playerPos.y}px);"
    on:mousedown={isMobile ? undefined : startPlayerDrag}
  >
    <div class="audio-top">
      <div class="audio-artwork">
        <img src={trackCover} alt="" />
        <div class="audio-artwork-overlay">
          {#if isPlaying}
            <div class="audio-bars">
              <span></span><span></span><span></span>
            </div>
          {/if}
        </div>
      </div>

      <div class="audio-info">
        <span class="audio-track">{trackName}</span>
        <span class="audio-artist">{artistName}</span>
      </div>

      <div class="audio-controls-play">
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <!-- svelte-ignore a11y-no-static-element-interactions -->
        <button class="audio-btn skip-btn" on:click={prevTrack}>
          <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor">
            <path d="M7 6v12h2V6H7zm10 0l-7 6 7 6V6z" />
          </svg>
        </button>

        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <!-- svelte-ignore a11y-no-static-element-interactions -->
        <button class="audio-btn play-btn" on:click={toggleAudio}>
          {#if isPlaying}
            <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor">
              <rect x="6" y="4" width="4" height="16" rx="1" />
              <rect x="14" y="4" width="4" height="16" rx="1" />
            </svg>
          {:else}
            <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor">
              <path d="M8 5v14l11-7z" />
            </svg>
          {/if}
        </button>

        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <!-- svelte-ignore a11y-no-static-element-interactions -->
        <button class="audio-btn skip-btn" on:click={nextTrack}>
          <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor">
            <path d="M7 6v12l7-6-7-6zm10 0v12h-2V6h2z" />
          </svg>
        </button>
      </div>
    </div>

    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
      class="audio-progress-container"
      on:mousedown={startProgressDrag}
      on:touchstart={handleProgressTouch}
    >
      <div class="audio-time">{formatTime(audioCurrentTime)}</div>
      <div class="audio-progress">
        <div class="audio-progress-bar" style="width: {audioProgress}%"></div>
        <div class="progress-knob" style="left: {audioProgress}%"></div>
      </div>
      <div class="audio-time">{formatTime(audioDuration)}</div>
    </div>

    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="audio-volume-row">
      <svg
        width="14"
        height="14"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
      >
        <path d="M11 5L6 9H2v6h4l5 4V5z" />
      </svg>
      <div
        class="volume-slider"
        on:mousedown={startVolumeDrag}
        on:touchstart={handleVolumeTouch}
      >
        <div class="volume-fill" style="width: {audioVolume * 100}%"></div>
        <div class="volume-knob" style="left: {audioVolume * 100}%"></div>
      </div>
      <svg
        width="14"
        height="14"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
      >
        <path d="M11 5L6 9H2v6h4l5 4V5z" />
        <path d="M19.07 4.93a10 10 0 010 14.14M15.54 8.46a5 5 0 010 7.07" />
      </svg>
    </div>
  </div>
{/if}

<!-- Particles -->
{#each particles as particle (particle.id)}
  <div
    class="particle"
    style="
      left: {particle.x}px;
      top: {particle.y}px;
      opacity: {particle.life};
      transform: scale({particle.life});
    "
  ></div>
{/each}

<main>
  <!-- Dual background videos for theme crossfade -->
  <div class="background">
    <video
      autoplay
      muted
      loop={true}
      playsinline
      class="bg-video"
      class:hidden-bg={isDarkTheme}
    >
      <source src="{base}bg.mp4" type="video/mp4" />
    </video>
    <video
      autoplay
      muted
      loop={true}
      playsinline
      class="bg-video bg-video-dark"
      class:visible-bg={isDarkTheme}
    >
      <source src="{base}blackbackground.mp4" type="video/mp4" />
    </video>
  </div>

  <canvas bind:this={canvasEl} class="visualizer-canvas"></canvas>

  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <!-- 3D Tilt Wrapper -->
  <div
    class="window-tilt-wrapper"
    style="transform: {isMobile
      ? 'none'
      : `perspective(1200px) rotateX(${shadowY * -0.6}deg) rotateY(${shadowX * 0.6}deg)`};"
  >
    <div
      class="window-card"
      class:show
      class:minimized
      class:closed
      class:dragging={isDragging}
      bind:this={windowEl}
      on:contextmenu={handleContextMenu}
      style="
        transform: translate({position.x}px, {position.y}px) {minimized
        ? 'scale(0.01) translateY(500px)'
        : closed
          ? 'scale(0.8)'
          : show
            ? ''
            : 'translateY(20px)'};
        box-shadow: {shadowX}px {shadowY +
        20}px 60px rgba(0, 0, 0, 0.4), {shadowX * 0.5}px {shadowY * 0.5 +
        10}px 30px rgba(0, 0, 0, 0.3);
      "
    >
      <div
        class="glare"
        style="transform: translate({shadowX * -0.6}px, {shadowY *
          -0.6}px); opacity: {Math.abs(shadowX) > 2 || Math.abs(shadowY) > 2
          ? 0.3
          : 0.1}"
      ></div>
      <!-- svelte-ignore a11y-no-static-element-interactions -->
      <div
        class="title-bar"
        on:mousedown={startDrag}
        on:dblclick={handleTitleBarDoubleClick}
      >
        <div class="traffic-lights">
          <button class="dot red" on:click={handleClose} aria-label="Close"
          ></button>
          <button
            class="dot yellow"
            on:click={handleMinimize}
            aria-label="Minimize"
          ></button>
          <button class="dot green" on:click={handleZoom} aria-label="Zoom"
          ></button>
        </div>
        <div class="title-container">
          <span class="window-title">bio.app</span>
        </div>
      </div>
      <div class="title-bar-line"></div>

      <div class="content">
        <!-- svelte-ignore a11y-no-static-element-interactions -->
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <div class="avatar-container" on:click={handleAvatarClick}>
          <img
            src="{base}avatar.webp"
            alt="Avatar"
            class="avatar"
            class:glitch={isGlitching}
            draggable="false"
            style="transform: rotate({avatarTilt.x}deg) translateY({avatarTilt.y}px);"
          />
        </div>

        <h1 class="animated-gradient">haos1y</h1>
        <p class="subtitle">
          {typedText}<span class="text-cursor" class:hidden={!showCursor}
            >|</span
          >
        </p>

        <div class="links">
          {#each links as link}
            <a
              href={link.url}
              target="_blank"
              rel="noopener noreferrer"
              class="link-btn"
              on:click={handleLinkClick}
            >
              <img
                src="{base}{link.icon}.{link.iconExt ?? 'svg'}"
                alt=""
                class="icon"
              />
              <span class="btn-text">{link.title}</span>
              <span class="tooltip {link.tooltipPos}">{link.tooltip}</span>
              {#if link.icon === "telegram" && showBadge}
                <span class="badge">{badgeCount}</span>
              {/if}
            </a>
          {/each}
        </div>
      </div>
    </div>
  </div>
</main>

<!-- macOS Dock -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="dock">
  <!-- bio.app window -->
  <div class="dock-item" on:click={openBioApp}>
    <div class="dock-tooltip">bio.app</div>
    <img src="{base}bioapp.svg" alt="bio.app" class="dock-icon" />
    <div class="dock-dot" class:active={show}></div>
  </div>

  <!-- Audio Player toggle -->
  <div class="dock-item" on:click={togglePlayer}>
    <div class="dock-tooltip">Плеер</div>
    <svg
      class="dock-icon dock-svg-icon"
      viewBox="0 0 24 24"
      fill="currentColor"
    >
      <path
        d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 14.5v-9l6 4.5-6 4.5z"
      />
    </svg>
    <div class="dock-dot" class:active={showPlayer}></div>
  </div>
</div>

<!-- Theme toggle button (top right) -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<button
  class="theme-btn"
  class:dark={isDarkTheme}
  on:click={toggleTheme}
  aria-label="Toggle theme"
>
  <img src="{base}theme.svg" alt="theme" />
</button>

<!-- Terminal Easter Egg Overlay -->
{#if showTerminal}
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div class="terminal-overlay" class:error-state={terminalError}>
    <div class="terminal-content">
      {#each terminalLines as line}
        <div class="terminal-line">{line}</div>
      {/each}
      {#if terminalInput || !terminalError}
        <div class="terminal-line input-line">
          {terminalInput}<span class="terminal-cursor">█</span>
        </div>
      {/if}
    </div>
  </div>
{/if}

<style>
  :global(*) {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    cursor: none !important;
  }

  :global(:root) {
    --bg: #000;
    --text: #f5f5f7;
    --text-2: #797979;
  }

  :global(body) {
    font-family:
      "SF Pro Display",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-weight: 500;
    background: var(--bg);
    color: var(--text);
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    overflow: hidden;
    user-select: none;
    -webkit-user-select: none;
    cursor: none !important;
  }

  /* Custom cursor */
  .custom-cursor {
    position: fixed;
    pointer-events: none;
    z-index: 99999;
    transition: transform 0.1s ease-out;
    will-change: transform, left, top;
    opacity: 0;
  }

  .custom-cursor.ready {
    opacity: 1;
  }

  .custom-cursor img {
    width: 26px;
    height: 26px;
    display: block;
  }

  .custom-cursor.pointer img {
    width: 30px;
    height: 30px;
  }

  main {
    width: 100vw;
    height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
  }

  /* Particles */
  .particle {
    position: fixed;
    width: 6px;
    height: 6px;
    background: rgba(255, 255, 255, 0.8);
    border-radius: 50%;
    pointer-events: none;
    z-index: 1000;
  }

  /* Background */
  .background {
    position: fixed;
    inset: 0;
    overflow: hidden;
    pointer-events: none;
    z-index: 0;
  }

  .bg-video {
    position: absolute;
    top: 50%;
    left: 50%;
    min-width: 100%;
    min-height: 100%;
    width: auto;
    height: auto;
    transform: translate(-50%, -50%);
    object-fit: cover;
  }

  /* Window Card - Frosted Glass */
  .window-card {
    width: 600px;
    height: 560px;
    background: rgba(0, 0, 0, 0.5);
    border: 1px solid rgba(255, 255, 255, 0.06);
    border-radius: 9px;
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    overflow: hidden;
    position: relative;
    z-index: 10;
    opacity: 0;
    transition: opacity 0.5s cubic-bezier(0.16, 1, 0.3, 1);
  }

  .window-card.show {
    opacity: 1;
  }

  .window-card.dragging {
    transition: none;
  }

  .window-card.minimized,
  .window-card.closed {
    opacity: 0;
    transition: all 0.4s cubic-bezier(0.16, 1, 0.3, 1);
  }

  /* Title Bar */
  .title-bar {
    display: flex;
    flex-direction: row;
    align-items: center;
    height: 28px;
    background: rgba(32, 32, 32, 0.85);
    box-shadow:
      0px 0.5px 0px rgba(0, 0, 0, 0.1),
      0px 1px 0px rgba(0, 0, 0, 0.1);
    backdrop-filter: blur(15px);
    -webkit-backdrop-filter: blur(15px);
    border-radius: 9px 9px 0px 0px;
  }

  .title-bar-line {
    width: 100%;
    height: 1px;
    background: #2a2a2a;
  }

  .traffic-lights {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    padding: 0px 8px;
    gap: 8px;
    width: 68px;
    height: 12px;
    flex: none;
  }

  .dot {
    box-sizing: border-box;
    width: 12px;
    height: 12px;
    border: 0.5px solid #666666;
    border-radius: 100px;
    flex: none;
    transition: all 0.15s;
    position: relative;
  }

  .dot::before {
    content: "";
    position: absolute;
    top: -6px;
    left: -6px;
    right: -6px;
    bottom: -6px;
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

  .title-container {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    padding: 3px 10px;
    height: 22px;
    flex: 1;
    margin-right: 68px;
  }

  .window-title {
    font-family:
      "SF Pro",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-style: normal;
    font-weight: 590;
    font-size: 13px;
    line-height: 16px;
    display: flex;
    align-items: center;
    text-align: center;
    color: rgba(255, 255, 255, 0.85);
  }

  /* Content */
  .content {
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
    padding-top: 48px;
  }

  /* Avatar */
  .avatar-container {
    width: 120px;
    height: 120px;
    margin-bottom: 12px;
  }

  .avatar {
    width: 120px;
    height: 120px;
    border-radius: 1000px;
    object-fit: cover;
    transition: transform 0.3s ease;
  }

  .avatar:hover:not(.glitch) {
    transform: scale(1.05) !important;
  }

  /* Glitch effect */
  .avatar.glitch {
    animation: glitch 0.3s cubic-bezier(0.25, 0.46, 0.45, 0.94) infinite;
  }

  @keyframes glitch {
    0% {
      transform: translate(0);
      filter: hue-rotate(0deg);
    }
    10% {
      transform: translate(-5px, 2px);
      filter: hue-rotate(90deg);
    }
    20% {
      transform: translate(5px, -2px);
      filter: hue-rotate(180deg);
    }
    30% {
      transform: translate(-3px, -3px);
      filter: hue-rotate(270deg);
    }
    40% {
      transform: translate(3px, 3px);
      filter: hue-rotate(360deg) saturate(2);
    }
    50% {
      transform: translate(-2px, 1px);
      filter: hue-rotate(45deg) brightness(1.5);
    }
    60% {
      transform: translate(2px, -1px);
      filter: hue-rotate(135deg);
    }
    70% {
      transform: translate(-4px, 2px);
      filter: hue-rotate(225deg) contrast(1.5);
    }
    80% {
      transform: translate(4px, -2px);
      filter: hue-rotate(315deg);
    }
    90% {
      transform: translate(-1px, 1px);
      filter: hue-rotate(180deg);
    }
    100% {
      transform: translate(0);
      filter: hue-rotate(0deg);
    }
  }

  /* Text with animated gradient */
  h1.animated-gradient {
    font-family:
      "SF Pro Display",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-style: normal;
    font-weight: 500;
    font-size: 28px;
    line-height: 33px;
    letter-spacing: -0.02em;
    background: linear-gradient(
      90deg,
      rgba(255, 255, 255, 0.8) 0%,
      rgba(229, 229, 229, 0.8) 25%,
      rgba(255, 255, 255, 0.9) 50%,
      rgba(201, 201, 201, 0.8) 75%,
      rgba(255, 255, 255, 0.8) 100%
    );
    background-size: 200% 100%;
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    margin: 0 0 2px 0;
    animation: shimmer 3s ease-in-out infinite;
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

  .subtitle {
    font-family:
      "SF Pro Display",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-style: normal;
    font-weight: 500;
    font-size: 20px;
    line-height: 24px;
    color: #797979;
    margin: 0 0 30px 0;
    min-height: 24px;
  }

  .text-cursor {
    animation: blink 1s step-end infinite;
  }

  .text-cursor.hidden {
    opacity: 0;
  }

  @keyframes blink {
    0%,
    100% {
      opacity: 1;
    }
    50% {
      opacity: 0;
    }
  }

  /* Links */
  .links {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
  }

  .link-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    width: auto;
    min-width: 155px;
    padding: 0 16px;
    height: 40px;
    background: rgba(0, 0, 0, 0.2);
    backdrop-filter: blur(2px);
    -webkit-backdrop-filter: blur(2px);
    border: none;
    text-decoration: none;
    border-radius: 10px;
    transition: all 0.2s;
    position: relative;
    user-select: none;
    -webkit-user-select: none;
  }

  .link-btn:hover {
    background: rgba(0, 0, 0, 0.3);
    transform: scale(1.02);
  }

  .link-btn:active {
    transform: scale(0.98);
  }

  /* Tooltip - Apple style */
  .tooltip {
    position: absolute;
    left: 50%;
    background: rgba(40, 40, 40, 0.95);
    color: rgba(255, 255, 255, 0.9);
    padding: 6px 10px;
    border-radius: 4px;
    font-size: 11px;
    font-weight: 400;
    letter-spacing: 0.01em;
    white-space: nowrap;
    opacity: 0;
    pointer-events: none;
    transition:
      opacity 0.15s ease,
      transform 0.15s ease;
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    box-shadow:
      0 2px 8px rgba(0, 0, 0, 0.3),
      0 0 1px rgba(255, 255, 255, 0.1) inset;
    border: 0.5px solid rgba(255, 255, 255, 0.1);
  }

  .tooltip.top {
    bottom: 100%;
    transform: translateX(-50%) translateY(-6px);
  }

  .tooltip.top::after {
    content: "";
    position: absolute;
    top: 100%;
    left: 50%;
    transform: translateX(-50%);
    border: 4px solid transparent;
    border-top-color: rgba(40, 40, 40, 0.95);
  }

  .tooltip.bottom {
    top: 100%;
    transform: translateX(-50%) translateY(6px);
  }

  .tooltip.bottom::after {
    content: "";
    position: absolute;
    bottom: 100%;
    left: 50%;
    transform: translateX(-50%);
    border: 4px solid transparent;
    border-bottom-color: rgba(40, 40, 40, 0.95);
  }

  .link-btn:hover .tooltip.top {
    opacity: 1;
    transform: translateX(-50%) translateY(-8px);
  }

  .link-btn:hover .tooltip.bottom {
    opacity: 1;
    transform: translateX(-50%) translateY(8px);
  }

  .btn-text {
    font-family:
      "SF Pro Display",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-style: normal;
    font-weight: 500;
    font-size: 18px;
    line-height: 21px;
    background: linear-gradient(90deg, #bfbfbf 0%, #dfdfdf 50%, #999999 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    pointer-events: none;
  }
  .icon {
    width: 15px;
    height: 15px;
    flex-shrink: 0;
    pointer-events: none;
  }

  /* Notification Badge - Apple style */
  .badge {
    position: absolute;
    top: -5px;
    right: -5px;
    min-width: 16px;
    height: 16px;
    background: linear-gradient(180deg, #ff453a 0%, #d70015 100%);
    border-radius: 8px;
    font-size: 10px;
    font-weight: 600;
    color: #fff;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0 4px;
    box-shadow:
      0 1px 3px rgba(0, 0, 0, 0.3),
      inset 0 1px 0 rgba(255, 255, 255, 0.2);
    border: 1px solid rgba(0, 0, 0, 0.1);
    animation: badgeIn 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
    pointer-events: none;
  }

  @keyframes badgeIn {
    0% {
      transform: scale(0);
      opacity: 0;
    }
    70% {
      transform: scale(1.15);
    }
    100% {
      transform: scale(1);
      opacity: 1;
    }
  }

  /* Spotlight */
  .spotlight-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    z-index: 1000;
    display: flex;
    justify-content: center;
    padding-top: 15vh;
    animation: fadeIn 0.2s ease;
  }

  .spotlight-overlay.closing {
    animation: fadeOut 0.2s ease forwards;
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
    }
    to {
      opacity: 1;
    }
  }

  @keyframes fadeOut {
    from {
      opacity: 1;
    }
    to {
      opacity: 0;
    }
  }

  .spotlight {
    width: 560px;
    max-height: 400px;
    background: rgba(30, 30, 30, 0.95);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 25px 80px rgba(0, 0, 0, 0.5);
    animation: slideDown 0.25s cubic-bezier(0.16, 1, 0.3, 1);
  }

  .spotlight.closing {
    animation: slideUp 0.2s ease forwards;
  }

  @keyframes slideDown {
    from {
      opacity: 0;
      transform: translateY(-20px) scale(0.95);
    }
    to {
      opacity: 1;
      transform: translateY(0) scale(1);
    }
  }

  @keyframes slideUp {
    from {
      opacity: 1;
      transform: translateY(0) scale(1);
    }
    to {
      opacity: 0;
      transform: translateY(-20px) scale(0.95);
    }
  }

  .spotlight-input-wrapper {
    display: flex;
    align-items: center;
    padding: 16px;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
    gap: 12px;
  }

  .spotlight-icon {
    color: #666;
    flex-shrink: 0;
  }

  .spotlight-input {
    flex: 1;
    background: none;
    border: none;
    outline: none;
    font-size: 18px;
    color: #fff;
    font-family: inherit;
  }

  .spotlight-input::placeholder {
    color: #666;
  }

  .spotlight-shortcut {
    font-size: 11px;
    color: #666;
    background: rgba(255, 255, 255, 0.1);
    padding: 4px 8px;
    border-radius: 4px;
    text-transform: uppercase;
  }

  .spotlight-results {
    max-height: 320px;
    overflow-y: auto;
    scrollbar-width: thin;
    scrollbar-color: rgba(255, 255, 255, 0.2) transparent;
  }

  .spotlight-results::-webkit-scrollbar {
    width: 6px;
  }

  .spotlight-results::-webkit-scrollbar-track {
    background: transparent;
  }

  .spotlight-results::-webkit-scrollbar-thumb {
    background: rgba(255, 255, 255, 0.2);
    border-radius: 3px;
  }

  .spotlight-results::-webkit-scrollbar-thumb:hover {
    background: rgba(255, 255, 255, 0.3);
  }

  .spotlight-item {
    display: flex;
    align-items: center;
    padding: 12px 16px;
    gap: 12px;
    transition: background 0.15s;
  }

  .spotlight-item:hover {
    background: rgba(255, 255, 255, 0.05);
  }

  .spotlight-item-icon {
    width: 32px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #999;
    flex-shrink: 0;
  }

  .spotlight-item-icon img {
    opacity: 0.7;
  }

  .spotlight-item-content {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  .spotlight-item-title {
    font-size: 14px;
    color: #fff;
  }

  .spotlight-item-desc {
    font-size: 12px;
    color: #666;
  }

  .spotlight-empty {
    padding: 24px;
    text-align: center;
    color: #666;
    font-size: 14px;
  }

  /* Keyboard hint */
  .keyboard-hint {
    position: fixed;
    top: 24px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 14px;
    background: rgba(0, 0, 0, 0.6);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 8px;
    z-index: 500;
    opacity: 1;
    transition:
      opacity 0.3s,
      transform 0.2s;
    color: #666;
  }

  .keyboard-hint svg {
    flex-shrink: 0;
  }

  .keyboard-hint:hover {
    background: rgba(0, 0, 0, 0.8);
    transform: translateX(-50%) scale(1.05);
  }

  .keyboard-hint:active {
    transform: translateX(-50%) scale(0.98);
  }

  .keyboard-hint.hidden {
    opacity: 0;
    pointer-events: none;
  }

  .hint-keys {
    display: flex;
    gap: 4px;
  }

  .key {
    font-size: 12px;
    color: #999;
    background: rgba(255, 255, 255, 0.1);
    padding: 4px 8px;
    border-radius: 4px;
    font-family:
      "SF Pro",
      -apple-system,
      sans-serif;
  }

  .hint-text {
    font-size: 12px;
    color: #666;
    margin-left: 4px;
  }

  /* Audio player - Apple Music style */
  .audio-player {
    position: fixed;
    bottom: 24px;
    right: 24px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 12px;
    background: rgba(20, 20, 20, 0.85);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    border: 0.5px solid rgba(255, 255, 255, 0.1);
    border-radius: 12px;
    z-index: 500;
    width: 260px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
  }

  .audio-top {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .audio-artwork {
    width: 44px;
    height: 44px;
    border-radius: 6px;
    overflow: hidden;
    position: relative;
    flex-shrink: 0;
  }

  .audio-artwork img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .audio-artwork-overlay {
    position: absolute;
    inset: 0;
    background: rgba(0, 0, 0, 0.3);
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0;
    transition: opacity 0.2s;
  }

  .audio-player.playing .audio-artwork-overlay {
    opacity: 1;
  }

  .audio-bars {
    display: flex;
    align-items: flex-end;
    gap: 2px;
    height: 16px;
  }

  .audio-bars span {
    width: 3px;
    background: #fff;
    border-radius: 1px;
    animation: audioBar 0.5s ease-in-out infinite alternate;
  }

  .audio-bars span:nth-child(1) {
    height: 8px;
    animation-delay: 0s;
  }
  .audio-bars span:nth-child(2) {
    height: 14px;
    animation-delay: 0.15s;
  }
  .audio-bars span:nth-child(3) {
    height: 10px;
    animation-delay: 0.3s;
  }

  @keyframes audioBar {
    to {
      height: 4px;
    }
  }

  .audio-info {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  .audio-track {
    font-size: 13px;
    font-weight: 500;
    color: #fff;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .audio-artist {
    font-size: 11px;
    color: rgba(255, 255, 255, 0.5);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .audio-controls {
    display: flex;
    align-items: center;
    gap: 4px;
  }

  .audio-controls-play {
    display: flex;
    align-items: center;
    gap: 4px;
    flex-shrink: 0;
  }

  .audio-btn {
    width: 32px;
    height: 32px;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.1);
    border: none;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #fff;
    transition:
      background 0.15s,
      transform 0.15s;
  }

  .skip-btn {
    width: 28px;
    height: 28px;
    background: transparent;
  }

  .skip-btn svg {
    width: 16px;
    height: 16px;
  }

  .audio-btn:hover {
    background: rgba(255, 255, 255, 0.2);
    transform: scale(1.05);
  }

  .audio-btn:active {
    transform: scale(0.95);
  }

  .play-btn {
    flex-shrink: 0;
  }

  .audio-volume-row {
    display: flex;
    align-items: center;
    gap: 8px;
    color: rgba(255, 255, 255, 0.4);
  }

  .volume-slider {
    flex: 1;
    height: 4px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 2px;
    position: relative;
  }

  .volume-fill {
    height: 100%;
    background: rgba(255, 255, 255, 0.8);
    border-radius: 2px;
  }

  .volume-knob {
    position: absolute;
    top: 50%;
    width: 12px;
    height: 12px;
    background: #fff;
    border-radius: 50%;
    transform: translate(-50%, -50%);
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.3);
    opacity: 0;
    transition: opacity 0.15s;
  }

  .audio-volume-row:hover .volume-knob {
    opacity: 1;
  }

  .audio-progress-container {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .audio-time {
    font-size: 10px;
    color: rgba(255, 255, 255, 0.4);
    min-width: 32px;
    font-variant-numeric: tabular-nums;
  }

  .audio-time:last-child {
    text-align: right;
  }

  .audio-progress {
    flex: 1;
    height: 4px;
    background: rgba(255, 255, 255, 0.2);
    border-radius: 2px;
    overflow: visible;
    position: relative;
  }

  .audio-progress-bar {
    height: 100%;
    background: #fff;
    border-radius: 2px;
  }

  .progress-knob {
    position: absolute;
    top: 50%;
    width: 12px;
    height: 12px;
    background: #fff;
    border-radius: 50%;
    transform: translate(-50%, -50%);
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.3);
    opacity: 0;
    transition: opacity 0.15s;
  }

  .audio-progress-container:hover .progress-knob {
    opacity: 1;
  }

  /* Context Menu - macOS style */
  .context-menu {
    position: fixed;
    min-width: 200px;
    background: rgba(30, 30, 30, 0.98);
    border: 0.5px solid rgba(255, 255, 255, 0.15);
    border-radius: 6px;
    padding: 4px 0;
    z-index: 2000;
    box-shadow:
      0 10px 40px rgba(0, 0, 0, 0.5),
      0 0 1px rgba(255, 255, 255, 0.1) inset;
    backdrop-filter: blur(30px);
    -webkit-backdrop-filter: blur(30px);
    animation: contextMenuIn 0.15s cubic-bezier(0.16, 1, 0.3, 1);
  }

  @keyframes contextMenuIn {
    from {
      opacity: 0;
      transform: scale(0.95);
    }
    to {
      opacity: 1;
      transform: scale(1);
    }
  }

  .context-menu-item {
    display: flex;
    align-items: center;
    padding: 6px 12px;
    gap: 10px;
    transition: background 0.1s;
    border-radius: 4px;
    margin: 0 4px;
  }

  .context-menu-item:hover {
    background: rgba(59, 130, 246, 0.8);
  }

  .context-menu-icon {
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: rgba(255, 255, 255, 0.7);
    flex-shrink: 0;
  }

  .context-menu-icon img {
    opacity: 0.7;
  }

  .context-menu-item:hover .context-menu-icon {
    color: #fff;
  }

  .context-menu-item:hover .context-menu-icon img {
    opacity: 1;
    filter: brightness(1.3);
  }

  .context-menu-label {
    flex: 1;
    font-size: 13px;
    color: rgba(255, 255, 255, 0.9);
  }

  .context-menu-shortcut {
    font-size: 12px;
    color: rgba(255, 255, 255, 0.4);
    font-family:
      "SF Pro",
      -apple-system,
      sans-serif;
  }

  .context-menu-item:hover .context-menu-shortcut {
    color: rgba(255, 255, 255, 0.7);
  }

  .context-menu-separator {
    height: 1px;
    background: rgba(255, 255, 255, 0.1);
    margin: 4px 12px;
  }

  /* Notifications - macOS style */
  .notifications {
    position: fixed;
    top: 16px;
    right: 16px;
    display: flex;
    flex-direction: column;
    gap: 8px;
    z-index: 3000;
    pointer-events: none;
  }

  .notification {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 12px 16px;
    background: rgba(40, 40, 40, 0.9);
    border: 0.5px solid rgba(255, 255, 255, 0.15);
    border-radius: 12px;
    backdrop-filter: blur(30px) saturate(180%);
    -webkit-backdrop-filter: blur(30px) saturate(180%);
    box-shadow:
      0 8px 32px rgba(0, 0, 0, 0.4),
      0 0 1px rgba(255, 255, 255, 0.1) inset;
    animation: notificationIn 0.4s cubic-bezier(0.16, 1, 0.3, 1);
    pointer-events: auto;
    min-width: 280px;
    max-width: 340px;
  }

  .notification.hiding {
    animation: notificationOut 0.3s ease forwards;
  }

  @keyframes notificationIn {
    from {
      opacity: 0;
      transform: translateX(100%) scale(0.9);
    }
    to {
      opacity: 1;
      transform: translateX(0) scale(1);
    }
  }

  @keyframes notificationOut {
    from {
      opacity: 1;
      transform: translateX(0) scale(1);
    }
    to {
      opacity: 0;
      transform: translateX(100%) scale(0.9);
    }
  }

  .notification-icon {
    width: 32px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    color: rgba(255, 255, 255, 0.8);
  }

  .notification-icon img {
    opacity: 0.9;
  }

  .notification-content {
    display: flex;
    flex-direction: column;
    gap: 2px;
    flex: 1;
    min-width: 0;
  }

  .notification-title {
    font-size: 13px;
    font-weight: 600;
    color: #fff;
  }

  .notification-message {
    font-size: 12px;
    color: rgba(255, 255, 255, 0.6);
    display: flex;
    align-items: center;
    gap: 4px;
  }

  .notif-key {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 10px;
    font-weight: 500;
    color: rgba(255, 255, 255, 0.8);
    background: rgba(255, 255, 255, 0.15);
    padding: 2px 5px;
    border-radius: 3px;
    font-family:
      "SF Pro",
      -apple-system,
      sans-serif;
  }

  /* Dual Background videos */
  .bg-video {
    transition: opacity 1.5s ease;
  }
  .bg-video.hidden-bg {
    opacity: 0;
  }
  .bg-video-dark {
    position: absolute;
    top: 50%;
    left: 50%;
    min-width: 100%;
    min-height: 100%;
    width: auto;
    height: auto;
    transform: translate(-50%, -50%);
    object-fit: cover;
    opacity: 0;
    transition: opacity 1.5s ease;
  }

  .bg-video-dark.visible-bg {
    opacity: 1;
  }

  /* Audio Visualizer Canvas */
  .visualizer-canvas {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 1;
  }

  /* 3D Tilt Glassmorphism Wrapper */
  .window-tilt-wrapper {
    position: relative;
    z-index: 10;
    transform-style: preserve-3d;
    transition: transform 0.1s ease-out; /* Smooth lerping is mostly done by svelte mousemove speed, but this softens it */
  }

  /* Glare effect inside window */
  .glare {
    position: absolute;
    inset: -50%;
    background: radial-gradient(
      circle at center,
      rgba(255, 255, 255, 0.8) 0%,
      rgba(255, 255, 255, 0) 60%
    );
    pointer-events: none;
    z-index: 100;
    transition:
      opacity 0.3s,
      transform 0.1s ease-out;
    mix-blend-mode: overlay;
  }

  /* Window Card - Frosted Glass */
  .window-card {
    width: 600px;
    height: 560px;
    background: rgba(0, 0, 0, 0.5);
    border: 1px solid rgba(255, 255, 255, 0.06);
    border-radius: 9px;
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    overflow: hidden;
    position: relative;
    z-index: 10;
    opacity: 0;
    transition: opacity 0.5s cubic-bezier(0.16, 1, 0.3, 1);
  }

  .window-card.show {
    opacity: 1;
  }

  /* Theme Button */
  .theme-btn {
    position: fixed;
    top: 24px;
    right: 24px;
    width: 44px;
    height: 44px;
    background: rgba(40, 40, 40, 0.4);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: none !important;
    z-index: 500;
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    transition: all 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  }
  .theme-btn:hover {
    transform: scale(1.1);
    background: rgba(60, 60, 60, 0.6);
  }
  .theme-btn:active {
    transform: scale(0.95);
  }
  .theme-btn img {
    width: 20px;
    height: 20px;
    transition: transform 0.5s cubic-bezier(0.16, 1, 0.3, 1);
    filter: invert(1);
  }
  .theme-btn.dark img {
    transform: rotate(180deg);
  }

  /* macOS Dock */
  .dock {
    position: fixed;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    align-items: flex-end;
    gap: 10px;
    padding: 8px 12px;
    background: rgba(30, 30, 30, 0.5);
    border: 0.5px solid rgba(255, 255, 255, 0.15);
    border-radius: 20px;
    backdrop-filter: blur(25px) saturate(180%);
    -webkit-backdrop-filter: blur(25px) saturate(180%);
    z-index: 1000;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  }

  .dock-item {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-end;
    cursor: none !important;
    transition: all 0.2s cubic-bezier(0.25, 1.2, 0.5, 1);
    outline: none;
    text-decoration: none;
  }

  .dock-icon {
    width: 36px;
    height: 36px;
    object-fit: contain;
    transition: all 0.2s cubic-bezier(0.25, 1.2, 0.5, 1);
    filter: drop-shadow(0 4px 6px rgba(0, 0, 0, 0.3));
    /* Provide a default background for SVG to make it stand out like an icon */
  }

  .dock-svg-icon {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 12px;
    padding: 8px;
    box-sizing: border-box;
    color: white;
  }

  .dock-item:hover {
    transform: translateY(-8px);
  }

  .dock-item:hover .dock-icon {
    transform: scale(1.3);
    filter: drop-shadow(0 10px 12px rgba(0, 0, 0, 0.5));
  }

  .dock-dot {
    width: 4px;
    height: 4px;
    background: rgba(255, 255, 255, 0.8);
    border-radius: 50%;
    margin-top: 4px;
    opacity: 0;
    transition: opacity 0.2s;
  }

  .dock-dot.active {
    opacity: 1;
  }

  .dock-tooltip {
    position: absolute;
    top: -38px;
    background: rgba(40, 40, 40, 0.9);
    color: rgba(255, 255, 255, 0.9);
    padding: 4px 10px;
    border-radius: 6px;
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.01em;
    white-space: nowrap;
    opacity: 0;
    pointer-events: none;
    transform: translateY(10px);
    transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    border: 0.5px solid rgba(255, 255, 255, 0.1);
  }

  .dock-tooltip::after {
    content: "";
    position: absolute;
    top: 100%;
    left: 50%;
    transform: translateX(-50%);
    border: 5px solid transparent;
    border-top-color: rgba(40, 40, 40, 0.9);
  }

  .dock-item:hover .dock-tooltip {
    opacity: 1;
    transform: translateY(0);
  }

  /* Terminal Easter Egg */
  .terminal-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.95);
    z-index: 9999;
    font-family: "Courier New", Courier, monospace;
    color: #4af626;
    padding: 40px;
    font-size: 16px;
    line-height: 1.5;
    overflow-y: auto;
    pointer-events: auto;
  }

  .terminal-overlay.error-state {
    background: #ff0000;
    color: #ffffff;
    font-weight: bold;
    animation: screenShake 0.4s cubic-bezier(0.36, 0.07, 0.19, 0.97) both
      infinite;
  }

  .terminal-content {
    max-width: 800px;
    margin: 0 auto;
    width: 100%;
  }

  .terminal-line {
    margin-bottom: 8px;
    white-space: pre-wrap;
    word-break: break-all;
  }

  .terminal-cursor {
    animation: terminalBlink 1s step-end infinite;
    display: inline-block;
    margin-left: 4px;
  }

  .error-state .terminal-cursor {
    display: none;
  }

  @keyframes terminalBlink {
    0%,
    100% {
      opacity: 1;
    }
    50% {
      opacity: 0;
    }
  }

  @keyframes screenShake {
    10%,
    90% {
      transform: translate3d(-2px, 2px, 0);
    }
    20%,
    80% {
      transform: translate3d(4px, -2px, 0);
    }
    30%,
    50%,
    70% {
      transform: translate3d(-6px, 4px, 0);
    }
    40%,
    60% {
      transform: translate3d(6px, -4px, 0);
    }
  }

  /* =============================================
     MOBILE RESPONSIVE STYLES (max-width: 767px)
     ============================================= */
  @media (max-width: 767px) {
    /* Hide custom cursor completely on touch devices */
    .custom-cursor {
      display: none !important;
    }

    /* Allow body to scroll on mobile */
    :global(body) {
      overflow: auto;
      cursor: auto !important;
    }

    :global(*) {
      cursor: auto !important;
    }

    /* Main layout — center window on mobile */
    main {
      align-items: center;
      justify-content: center;
      padding: 16px 0 130px 0;
      min-height: 100vh;
      height: auto;
    }

    /* Window tilt wrapper — centered block */
    .window-tilt-wrapper {
      transform: none !important;
      width: calc(100vw - 24px);
      max-width: 480px;
      margin: 0 auto;
      display: block;
      padding: 0;
    }

    /* Window card — fill wrapper */
    .window-card {
      width: 100% !important;
      max-width: none;
      height: auto !important;
      min-height: 0;
      overflow: visible;
    }

    /* Content padding */
    .content {
      padding-top: 28px;
      padding-bottom: 24px;
    }

    /* Avatar */
    .avatar-container {
      width: 90px;
      height: 90px;
      margin-bottom: 10px;
    }

    .avatar {
      width: 90px;
      height: 90px;
    }

    /* Title */
    h1.animated-gradient {
      font-size: 22px;
      line-height: 26px;
    }

    /* Subtitle */
    .subtitle {
      font-size: 14px;
      line-height: 18px;
      margin: 0 12px 20px 12px;
    }

    /* Links */
    .links {
      width: 100%;
      padding: 0 16px;
      align-items: stretch;
    }

    .link-btn {
      width: 100%;
      min-width: 0;
      height: 46px;
      font-size: 16px;
      justify-content: flex-start;
      padding: 0 20px;
    }

    .btn-text {
      font-size: 16px;
    }

    .icon {
      width: 18px;
      height: 18px;
    }

    /* Hide tooltip on mobile (no hover) */
    .tooltip {
      display: none;
    }

    /* Audio player — centered at bottom, full width */
    .audio-player {
      left: 50% !important;
      right: auto !important;
      bottom: 72px !important;
      transform: translateX(-50%) !important;
      width: min(92vw, 340px) !important;
    }

    /* Show volume/progress knobs always on mobile (no hover) */
    .volume-knob,
    .progress-knob {
      opacity: 1 !important;
    }

    /* Make progress and volume bars taller for easier touch */
    .audio-progress,
    .volume-slider {
      height: 6px;
    }

    /* Dock — smaller, more compact */
    .dock {
      padding: 6px 10px;
      gap: 8px;
      bottom: 14px;
    }

    .dock-icon {
      width: 30px;
      height: 30px;
    }

    .dock-svg-icon {
      padding: 6px;
    }

    /* Hide keyboard hint on mobile (no keyboard shortcuts) */
    .keyboard-hint {
      display: none !important;
    }

    /* Notifications — full width aware */
    .notifications {
      right: 8px;
      top: 8px;
      max-width: calc(100vw - 16px);
    }

    .notification {
      min-width: 0;
      max-width: calc(100vw - 16px);
    }

    /* Spotlight — full width on mobile */
    .spotlight-overlay {
      padding-top: 10vh;
      align-items: flex-start;
    }

    .spotlight {
      width: 92vw;
      max-width: none;
    }

    /* Theme button — keep it accessible */
    .theme-btn {
      top: 12px;
      right: 12px;
      cursor: auto !important;
    }

    /* Terminal overlay — better font size on small screens */
    .terminal-overlay {
      padding: 20px 16px;
      font-size: 13px;
    }
  }
</style>
