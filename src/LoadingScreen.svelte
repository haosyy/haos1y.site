<script>
  import { createEventDispatcher } from 'svelte';
  
  let visible = true;
  let fadeOut = false;
  let ready = false;
  let mouseX = 0;
  let mouseY = 0;
  
  const dispatch = createEventDispatcher();
  
  setTimeout(() => {
    ready = true;
  }, 1000);
  
  function handleMove(e) {
    mouseX = e.clientX;
    mouseY = e.clientY;
  }
  
  function handleClick() {
    if (!ready) return;
    fadeOut = true;
    dispatch('enter');
    setTimeout(() => { 
      visible = false;
    }, 600);
  }
</script>

{#if visible}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div class="loader" class:fadeOut class:ready on:mousemove={handleMove} on:click={handleClick}>
    {#if !ready}
      <div class="ispinner">
        {#each Array(8) as _}
          <div class="ispinner-blade"></div>
        {/each}
      </div>
    {/if}
    
    {#if ready}
      <div class="cursor-hint" style="left: {mouseX}px; top: {mouseY}px;">
        <span>enter</span>
      </div>
    {/if}
  </div>
{/if}

<style>
  .loader {
    position: fixed;
    inset: 0;
    z-index: 9999;
    background: #000;
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 1;
    transition: opacity 0.6s ease;
  }
  
  .loader.ready {
    cursor: none;
  }
  
  .loader.fadeOut {
    opacity: 0;
    pointer-events: none;
  }
  
  .ispinner {
    position: relative;
    width: 20px;
    height: 20px;
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
  
  .ispinner-blade:nth-child(1) { transform: rotate(0deg); animation-delay: -0.875s; }
  .ispinner-blade:nth-child(2) { transform: rotate(45deg); animation-delay: -0.75s; }
  .ispinner-blade:nth-child(3) { transform: rotate(90deg); animation-delay: -0.625s; }
  .ispinner-blade:nth-child(4) { transform: rotate(135deg); animation-delay: -0.5s; }
  .ispinner-blade:nth-child(5) { transform: rotate(180deg); animation-delay: -0.375s; }
  .ispinner-blade:nth-child(6) { transform: rotate(225deg); animation-delay: -0.25s; }
  .ispinner-blade:nth-child(7) { transform: rotate(270deg); animation-delay: -0.125s; }
  .ispinner-blade:nth-child(8) { transform: rotate(315deg); animation-delay: 0s; }
  
  @keyframes blade {
    0% { opacity: 1; }
    100% { opacity: 0.15; }
  }
  
  .cursor-hint {
    position: fixed;
    pointer-events: none;
    transform: translate(-50%, -50%);
    width: 80px;
    height: 80px;
    border-radius: 50%;
    background: rgba(255, 255, 255, 0.08);
    display: flex;
    align-items: center;
    justify-content: center;
    opacity: 0;
    animation: cursorIn 0.4s ease forwards;
    transition: width 0.2s, height 0.2s;
  }
  
  .cursor-hint span {
    font-family: 'SF Pro Display', -apple-system, BlinkMacSystemFont, sans-serif;
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: rgba(255, 255, 255, 0.6);
  }
  
  @keyframes cursorIn {
    from {
      opacity: 0;
      transform: translate(-50%, -50%) scale(0.5);
    }
    to {
      opacity: 1;
      transform: translate(-50%, -50%) scale(1);
    }
  }
</style>
