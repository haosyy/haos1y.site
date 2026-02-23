<script>
  import { createEventDispatcher } from "svelte";

  let visible = true;
  let fadeOut = false;
  let ready = false;
  let loading = false;
  let progress = 0;
  let mouseX = 0;
  let mouseY = 0;

  const isMobile =
    typeof window !== "undefined" &&
    ("ontouchstart" in window ||
      navigator.maxTouchPoints > 0 ||
      window.innerWidth < 768);

  const dispatch = createEventDispatcher();

  // Show spinner for a moment, then become ready
  setTimeout(() => {
    ready = true;
    if (isMobile) {
      // Mobile: auto-start progress bar
      startLoading();
    }
  }, 800);

  function handleMove(e) {
    mouseX = e.clientX;
    mouseY = e.clientY;
  }

  function handleClick() {
    if (!ready || loading) return;
    startLoading();
  }

  function startLoading() {
    loading = true;
    progress = 0;

    // Animate progress 0→100 with ease-out curve over ~2s
    const duration = 2000;
    const steps = 80;
    const stepTime = duration / steps;
    let step = 0;

    const interval = setInterval(() => {
      step++;
      // Ease-out: fast at start, decelerate near 100%
      const t = step / steps;
      progress = Math.min(100, (1 - Math.pow(1 - t, 2.2)) * 100);

      if (step >= steps) {
        clearInterval(interval);
        // Brief pause at 100%, then dispatch enter and fade
        setTimeout(() => {
          dispatch("enter");
          fadeOut = true;
          setTimeout(() => {
            visible = false;
          }, 550);
        }, 180);
      }
    }, stepTime);
  }
</script>

{#if visible}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div
    class="loader"
    class:fadeOut
    on:mousemove={handleMove}
    on:click={handleClick}
  >
    {#if !loading}
      <!-- Phase 1: spinner / "enter" hint -->
      <div class="phase-initial">
        {#if !ready}
          <div class="ispinner">
            {#each Array(8) as _}
              <div class="ispinner-blade"></div>
            {/each}
          </div>
        {/if}

        {#if ready && !isMobile}
          <div class="cursor-hint" style="left: {mouseX}px; top: {mouseY}px;">
            <span>enter</span>
          </div>
        {/if}
      </div>
    {:else}
      <!-- Phase 2: progress bar -->
      <div class="phase-progress">
        <div class="brand">
          <span class="brand-name">haos1y</span>
          <span class="brand-sub">bio.app</span>
        </div>

        <div class="progress-wrap">
          <div class="progress-track">
            <div class="progress-fill" style="width: {progress}%"></div>
          </div>
          <span class="progress-pct">{Math.floor(progress)}%</span>
        </div>
      </div>
    {/if}
  </div>
{/if}

<style>
  .loader {
    position: fixed;
    inset: 0;
    z-index: 99999;
    background: #000;
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 1;
    transition: opacity 0.55s ease;
  }

  .loader.fadeOut {
    opacity: 0;
    pointer-events: none;
  }

  /* ── Phase 1: spinner ── */
  .phase-initial {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 100%;
  }

  .ispinner {
    position: relative;
    width: 22px;
    height: 22px;
  }

  .ispinner-blade {
    position: absolute;
    width: 2px;
    height: 5px;
    background: #8e8e93;
    left: 50%;
    top: 50%;
    margin-left: -1px;
    margin-top: -8px;
    border-radius: 1px;
    transform-origin: center 8px;
    animation: blade 1s linear infinite;
  }

  .ispinner-blade:nth-child(1) {
    transform: rotate(0deg);
    animation-delay: -0.875s;
  }
  .ispinner-blade:nth-child(2) {
    transform: rotate(45deg);
    animation-delay: -0.75s;
  }
  .ispinner-blade:nth-child(3) {
    transform: rotate(90deg);
    animation-delay: -0.625s;
  }
  .ispinner-blade:nth-child(4) {
    transform: rotate(135deg);
    animation-delay: -0.5s;
  }
  .ispinner-blade:nth-child(5) {
    transform: rotate(180deg);
    animation-delay: -0.375s;
  }
  .ispinner-blade:nth-child(6) {
    transform: rotate(225deg);
    animation-delay: -0.25s;
  }
  .ispinner-blade:nth-child(7) {
    transform: rotate(270deg);
    animation-delay: -0.125s;
  }
  .ispinner-blade:nth-child(8) {
    transform: rotate(315deg);
    animation-delay: 0s;
  }

  @keyframes blade {
    0% {
      opacity: 1;
    }
    100% {
      opacity: 0.15;
    }
  }

  /* Cursor follow hint (desktop only) */
  .cursor-hint {
    position: fixed;
    pointer-events: none;
    transform: translate(-50%, -50%);
    width: 78px;
    height: 78px;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.07);
    border: 1px solid rgba(255, 255, 255, 0.1);
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0;
    animation: hintIn 0.4s ease forwards;
  }

  .cursor-hint span {
    font-family:
      "SF Pro Display",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-size: 10px;
    font-weight: 600;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.55);
  }

  @keyframes hintIn {
    from {
      opacity: 0;
      transform: translate(-50%, -50%) scale(0.6);
    }
    to {
      opacity: 1;
      transform: translate(-50%, -50%) scale(1);
    }
  }

  /* ── Phase 2: progress bar ── */
  .phase-progress {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 28px;
    animation: progressIn 0.35s cubic-bezier(0.16, 1, 0.3, 1);
  }

  @keyframes progressIn {
    from {
      opacity: 0;
      transform: translateY(10px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  .brand {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
  }

  .brand-name {
    font-family:
      "SF Pro Display",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-size: 32px;
    font-weight: 600;
    letter-spacing: -0.02em;
    background: linear-gradient(
      90deg,
      rgba(255, 255, 255, 0.7) 0%,
      rgba(230, 230, 230, 0.85) 30%,
      #fff 50%,
      rgba(210, 210, 210, 0.8) 70%,
      rgba(255, 255, 255, 0.7) 100%
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

  .brand-sub {
    font-family:
      "SF Pro Display",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-size: 12px;
    font-weight: 500;
    letter-spacing: 0.1em;
    color: rgba(255, 255, 255, 0.3);
    text-transform: uppercase;
  }

  .progress-wrap {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    width: 240px;
  }

  .progress-track {
    width: 100%;
    height: 2px;
    background: rgba(255, 255, 255, 0.1);
    border-radius: 2px;
    overflow: hidden;
  }

  .progress-fill {
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

  .progress-pct {
    font-family:
      "SF Pro Display",
      -apple-system,
      BlinkMacSystemFont,
      sans-serif;
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.06em;
    color: rgba(255, 255, 255, 0.25);
    min-width: 34px;
    text-align: center;
    font-variant-numeric: tabular-nums;
  }
</style>
