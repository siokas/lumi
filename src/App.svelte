<script lang="ts">
  // --- Types ---
  interface Particle {
    x: number;
    y: number;
    originX: number;
    originY: number;
    vx: number;
    vy: number;
    size: number;
    color: string;
    angle: number;
  }

  interface BackgroundParticle {
    x: number;
    y: number;
    vx: number;
    vy: number;
    size: number;
    alpha: number;
    phase: number;
  }

  interface MouseState {
    x: number;
    y: number;
    isActive: boolean;
  }

  // --- Configuration Constants ---
  const PARTICLE_DENSITY = 0.00015; // Interactive foreground particle density
  const BG_PARTICLE_DENSITY = 0.00005; // Ambient background particle density
  const MOUSE_RADIUS = 180; // Radius of mouse repulsion
  const RETURN_SPEED = 0.08; // Return stiffness spring constant
  const DAMPING = 0.9; // Velocity decay factor (friction)
  const REPULSION_STRENGTH = 1.2; // Force multiplier when pushed by mouse

  const randomRange = (min: number, max: number) =>
    Math.random() * (max - min) + min;

  // --- Svelte 5 Reactive Elements and DOM Binds ---
  let canvasRef = $state<HTMLCanvasElement>();
  let containerRef = $state<HTMLDivElement>();
  let debugInfo = $state({ count: 0, fps: 0 });

  // --- Dark/Light Mode State ---
  let isLightMode = $state(false);
  let transitionOverlay = $state<HTMLDivElement>();
  let dotRef = $state<SVGPathElement>();
  let isTransitioning = $state(false);

  // --- Physics Engine State (Plain JS arrays/variables for maximum framerate) ---
  let particles: Particle[] = [];
  let backgroundParticles: BackgroundParticle[] = [];
  let mouse: MouseState = { x: -1000, y: -1000, isActive: false };
  let frameId = 0;
  let lastTime = 0;

  // Initialize Particles based on container size
  function initParticles(width: number, height: number) {
    // 1. Foreground Interactive Particles
    const particleCount = Math.floor(width * height * PARTICLE_DENSITY);
    const newParticles: Particle[] = [];

    for (let i = 0; i < particleCount; i++) {
      const x = Math.random() * width;
      const y = Math.random() * height;

      newParticles.push({
        x,
        y,
        originX: x,
        originY: y,
        vx: 0,
        vy: 0,
        size: randomRange(1, 2.5),
        color: Math.random() > 0.9 ? "#4285F4" : "#ffffff",
        angle: Math.random() * Math.PI * 2,
      });
    }
    particles = newParticles;

    // 2. Ambient Background Stars (Snow)
    const bgCount = Math.floor(width * height * BG_PARTICLE_DENSITY);
    const newBgParticles: BackgroundParticle[] = [];

    for (let i = 0; i < bgCount; i++) {
      newBgParticles.push({
        x: Math.random() * width,
        y: Math.random() * height,
        vx: (Math.random() - 0.5) * 0.05, // Gentle horizontal drift
        vy: randomRange(0.15, 0.45), // Constant gentle downward movement (snow fall)
        size: randomRange(0.5, 1.5),
        alpha: randomRange(0.1, 0.4),
        phase: Math.random() * Math.PI * 2,
      });
    }
    backgroundParticles = newBgParticles;

    // Update entity counter state
    debugInfo.count = particleCount + bgCount;
  }

  // Canvas Core Render Loop
  function animate(time: number) {
    if (!canvasRef) return;
    const ctx = canvasRef.getContext("2d");
    if (!ctx) return;

    // Calculate FPS
    const delta = time - lastTime;
    lastTime = time;
    if (delta > 0) {
      debugInfo.fps = Math.round(1000 / delta);
    }

    const dpr = window.devicePixelRatio || 1;
    const canvasWidth = canvasRef.width / dpr;
    const canvasHeight = canvasRef.height / dpr;

    // Clear with respect to the logical scale
    ctx.clearRect(0, 0, canvasWidth, canvasHeight);

    // --- Fill background based on theme ---
    ctx.fillStyle = isLightMode ? "#f0f2f5" : "#1C2329";
    ctx.fillRect(0, 0, canvasWidth, canvasHeight);

    // --- Render Part 1: Pulsating Deep Radial Glow ---
    const centerX = canvasWidth / 2;
    const centerY = canvasHeight / 2;
    const pulseSpeed = 0.0008;
    const pulseOpacity = Math.sin(time * pulseSpeed) * 0.035 + 0.085; // Oscillate opacity

    const glowGradient = ctx.createRadialGradient(
      centerX,
      centerY,
      0,
      centerX,
      centerY,
      Math.max(canvasWidth, canvasHeight) * 0.7,
    );

    if (isLightMode) {
      glowGradient.addColorStop(0, `rgba(37, 190, 194, ${pulseOpacity * 0.6})`);
      glowGradient.addColorStop(1, "rgba(240, 242, 245, 0)");
    } else {
      glowGradient.addColorStop(0, `rgba(66, 133, 244, ${pulseOpacity})`);
      glowGradient.addColorStop(1, "rgba(0, 0, 0, 0)");
    }

    ctx.fillStyle = glowGradient;
    ctx.fillRect(0, 0, canvasWidth, canvasHeight);

    // --- Render Part 2: Background Drifting Stars (Snow) ---
    ctx.fillStyle = isLightMode ? "#b0b8c4" : "#ffffff";
    for (let i = 0; i < backgroundParticles.length; i++) {
      const p = backgroundParticles[i];
      p.y += p.vy;
      p.x += p.vx + Math.sin(time * 0.001 + p.phase) * 0.08; // Subtle horizontal sway

      // Screen wrap borders for falling snow
      if (p.y > canvasHeight) {
        p.y = 0;
        p.x = Math.random() * canvasWidth;
      }
      if (p.x < 0) p.x = canvasWidth;
      if (p.x > canvasWidth) p.x = 0;

      // Twinkle luminance calculation
      const twinkle = Math.sin(time * 0.002 + p.phase) * 0.5 + 0.5;
      const currentAlpha = p.alpha * (0.3 + 0.7 * twinkle);

      ctx.globalAlpha = currentAlpha;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
      ctx.fill();
    }
    ctx.globalAlpha = 1.0; // Reset globalAlpha for foreground rendering

    // --- Physics Phase 1: Apply Mouse Repulsion and Return Springs ---
    for (let i = 0; i < particles.length; i++) {
      const p = particles[i];

      // Mouse repulsion calculations
      const dx = mouse.x - p.x;
      const dy = mouse.y - p.y;
      const distance = Math.max(Math.sqrt(dx * dx + dy * dy), 1); // Avoid division by zero

      if (mouse.isActive && distance < MOUSE_RADIUS) {
        const forceDirectionX = dx / distance;
        const forceDirectionY = dy / distance;
        const force = (MOUSE_RADIUS - distance) / MOUSE_RADIUS;

        const repulsion = force * REPULSION_STRENGTH;
        p.vx -= forceDirectionX * repulsion * 5;
        p.vy -= forceDirectionY * repulsion * 5;
      }

      // Anchor/Origin drift (snow falling effect)
      // Larger particles fall faster, keeping it very gentle ("not too much")
      const driftSpeed = 0.12 + p.size * 0.08;
      p.originY += driftSpeed;

      // Gentle horizontal sway on the origin anchor
      p.originX += Math.sin(time * 0.001 + p.angle) * 0.08;

      // Wrap-around screen bounds for origins and particles
      if (p.originY > canvasHeight) {
        p.originY = 0;
        p.originX = Math.random() * canvasWidth;
        p.x = p.originX;
        p.y = 0;
        p.vx = 0;
        p.vy = 0;
      }
      if (p.originX < 0) {
        p.originX = canvasWidth;
        p.x = canvasWidth;
      } else if (p.originX > canvasWidth) {
        p.originX = 0;
        p.x = 0;
      }

      // Return-to-origin Spring calculations
      const springDx = p.originX - p.x;
      const springDy = p.originY - p.y;

      p.vx += springDx * RETURN_SPEED;
      p.vy += springDy * RETURN_SPEED;
    }

    // --- Physics Phase 2: Inter-Particle Elastic Collisions (Circle-Circle) ---
    for (let i = 0; i < particles.length; i++) {
      for (let j = i + 1; j < particles.length; j++) {
        const p1 = particles[i];
        const p2 = particles[j];

        const dx = p2.x - p1.x;
        const dy = p2.y - p1.y;
        const distSq = dx * dx + dy * dy;
        const minDist = p1.size + p2.size;

        if (distSq < minDist * minDist) {
          const dist = Math.sqrt(distSq);
          if (dist > 0.01) {
            const nx = dx / dist; // Normal vector
            const ny = dy / dist;

            // Resolve overlapping (Static overlap push)
            const overlap = minDist - dist;
            const pushX = nx * overlap * 0.5;
            const pushY = ny * overlap * 0.5;

            p1.x -= pushX;
            p1.y -= pushY;
            p2.x += pushX;
            p2.y += pushY;

            // Elastic Collision velocities
            const dvx = p1.vx - p2.vx;
            const dvy = p1.vy - p2.vy;
            const velocityAlongNormal = dvx * nx + dvy * ny;

            // Bounce only if moving towards each other
            if (velocityAlongNormal > 0) {
              const m1 = p1.size; // Size used as mass representation
              const m2 = p2.size;
              const restitution = 0.85; // Elasticity constant

              const impulseMagnitude =
                (-(1 + restitution) * velocityAlongNormal) / (1 / m1 + 1 / m2);

              const impulseX = impulseMagnitude * nx;
              const impulseY = impulseMagnitude * ny;

              p1.vx += impulseX / m1;
              p1.vy += impulseY / m1;
              p2.vx -= impulseX / m2;
              p2.vy -= impulseY / m2;
            }
          }
        }
      }
    }

    // --- Physics Phase 3: Update Integration and Paint Interactive Particles ---
    for (let i = 0; i < particles.length; i++) {
      const p = particles[i];

      p.vx *= DAMPING;
      p.vy *= DAMPING;

      p.x += p.vx;
      p.y += p.vy;

      ctx.beginPath();
      ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);

      const velocity = Math.sqrt(p.vx * p.vx + p.vy * p.vy);
      const opacity = Math.min(0.3 + velocity * 0.1, 1.0);

      ctx.fillStyle = isLightMode
        ? (p.color === "#ffffff" ? `rgba(100, 116, 139, ${opacity})` : "#2563eb")
        : (p.color === "#ffffff" ? `rgba(255, 255, 255, ${opacity})` : p.color);

      ctx.fill();
    }

    frameId = requestAnimationFrame(animate);
  }

  // Mouse move and hover handler
  function handleMouseMove(e: MouseEvent) {
    if (!containerRef) return;
    const rect = containerRef.getBoundingClientRect();
    mouse = {
      x: e.clientX - rect.left,
      y: e.clientY - rect.top,
      isActive: true,
    };
  }

  // Mouse leave handler
  function handleMouseLeave() {
    mouse.isActive = false;
    mouse.x = -1000;
    mouse.y = -1000;
  }

  // --- Dark/Light Mode Toggle with Circular Reveal ---
  function handleDotClick(e: MouseEvent) {
    if (isTransitioning) return;
    isTransitioning = true;

    const dot = e.currentTarget as SVGPathElement;
    const rect = dot.getBoundingClientRect();
    const originX = rect.left + rect.width / 2;
    const originY = rect.top + rect.height / 2;

    // Calculate the maximum radius needed to cover the entire viewport
    const maxRadius = Math.ceil(
      Math.sqrt(
        Math.max(originX, window.innerWidth - originX) ** 2 +
        Math.max(originY, window.innerHeight - originY) ** 2
      )
    );

    if (transitionOverlay) {
      const overlay = transitionOverlay;

      // Paint the overlay with the CURRENT (old) theme's background color,
      // covering the full screen so it looks identical to the current state.
      overlay.style.backgroundColor = isLightMode ? "#f0f2f5" : "#1C2329";
      overlay.style.transition = "none";
      overlay.style.clipPath = `circle(${maxRadius}px at ${originX}px ${originY}px)`;
      overlay.style.display = "block";

      // Force reflow so the full-coverage clip-path is committed to the GPU
      overlay.offsetHeight;

      // Immediately flip the theme — the canvas and CSS underneath start
      // rendering in the new palette right away, but the overlay hides them.
      isLightMode = !isLightMode;

      // Now animate the OLD-theme overlay shrinking to circle(0), which
      // progressively reveals the live new-theme canvas expanding outward
      // from the dot.
      overlay.style.transition = `clip-path 1.2s cubic-bezier(0.4, 0, 0.2, 1)`;
      overlay.style.clipPath = `circle(0px at ${originX}px ${originY}px)`;

      const onEnd = () => {
        overlay.removeEventListener("transitionend", onEnd);
        overlay.style.transition = "none";
        overlay.style.display = "none";
        isTransitioning = false;
      };
      overlay.addEventListener("transitionend", onEnd);
    }
  }

  // Svelte 5 Reactive Effect for setup and resize handling
  $effect(() => {
    if (!containerRef || !canvasRef) return;

    const handleResize = () => {
      if (containerRef && canvasRef) {
        const { width, height } = containerRef.getBoundingClientRect();
        const dpr = window.devicePixelRatio || 1;

        canvasRef.width = width * dpr;
        canvasRef.height = height * dpr;

        canvasRef.style.width = `${width}px`;
        canvasRef.style.height = `${height}px`;

        const ctx = canvasRef.getContext("2d");
        if (ctx) {
          ctx.scale(dpr, dpr);
        }

        initParticles(width, height);
      }
    };

    window.addEventListener("resize", handleResize);
    handleResize();

    frameId = requestAnimationFrame(animate);

    return () => {
      window.removeEventListener("resize", handleResize);
      cancelAnimationFrame(frameId);
    };
  });
</script>

<div class="app-container" class:light-mode={isLightMode}>
  <!-- Circular Transition Overlay -->
  <div bind:this={transitionOverlay} class="theme-transition-overlay"></div>

  <!-- Interactive Physics Canvas Background -->
  <div
    bind:this={containerRef}
    class="canvas-container"
    class:light-mode={isLightMode}
    onmousemove={handleMouseMove}
    onmouseleave={handleMouseLeave}
    role="presentation"
  >
    <canvas bind:this={canvasRef}></canvas>
  </div>

  <!-- Hero Main Copy & Actions -->
  <main class="hero-content">
    <div class="hero-inner">
      <div class="tag-container animate-fade-in-up">
        <span class="hero-tag"> Educational Technology Specialists </span>
      </div>

      <div class="logo-container animate-fade-in-up-delay-1">
        <svg
          id="Layer_2"
          data-name="Layer 2"
          xmlns="http://www.w3.org/2000/svg"
          viewBox="0 0 588.71 340.77"
          class="hero-logo"
        >
          <defs>
            <style>
              .cls-1,
              .cls-2 {
                fill: #a0dade;
              }

              .cls-1,
              .cls-3 {
                fill-rule: evenodd;
              }

              .cls-4 {
                fill: #c7d8d8;
                font-family: GoogleSansFlex-Regular, "Google Sans Flex";
                font-size: 46.64px;
                font-variation-settings:
                  "opsz" 18,
                  "wdth" 100,
                  "wght" 400,
                  "GRAD" 0,
                  "ROND" 0,
                  "slnt" 0;
              }

              .cls-5 {
                letter-spacing: -0.03em;
              }

              .cls-3 {
                fill: #25bec2;
              }

              .cls-6 {
                letter-spacing: 0em;
              }
            </style>
          </defs>
          <g id="Layer_1-2" data-name="Layer 1">
            <g>
              <g>
                <g>
                  <path
                    class="cls-2"
                    d="M116.14,257.86H8.13c-5.09,0-7.64-2.55-7.64-7.64V112.72c0-5.09,2.55-7.64,7.64-7.64h32.89c5.09,0,7.64,2.55,7.64,7.64v105.04h29.07v-34.38c0-5.09,2.55-7.64,7.64-7.64h30.77c5.09,0,7.64,2.55,7.64,7.64v66.84c0,5.09-2.55,7.64-7.64,7.64Z"
                  />
                  <path
                    class="cls-2"
                    d="M278.05,215.42c0,6.37-.95,12.16-2.86,17.35-1.91,5.19-5.31,9.66-10.19,13.4-4.88,3.74-11.56,6.62-20.05,8.62-8.49,2.01-19.31,3.01-32.47,3.01s-23.98-1.01-32.47-3.01c-8.49-2.01-15.17-4.88-20.05-8.62-4.88-3.74-8.28-8.21-10.19-13.4s-2.86-10.98-2.86-17.35v-102.86c0-4.99,2.55-7.48,7.64-7.48h33.31c5.09,0,7.64,2.49,7.64,7.48v94.76c0,2.77,1.06,5.09,3.18,6.96,2.12,1.87,6.86,2.81,14.22,2.81s12.31-.93,14.43-2.81c2.12-1.87,3.18-4.19,3.18-6.96v-94.76c0-4.99,2.55-7.48,7.64-7.48h32.25c5.09,0,7.64,2.49,7.64,7.48v102.86Z"
                  />
                  <path
                    class="cls-2"
                    d="M386.48,146.03l18.89-34.59c2.26-4.24,5.52-6.37,9.76-6.37h34.38c5.09,0,7.64,2.55,7.64,7.64v137.5c0,5.09-2.55,7.64-7.64,7.64h-32.68c-5.09,0-7.64-2.55-7.64-7.64v-74.69l-12.52,23.13c-2.41,4.53-6.15,6.79-11.25,6.79h-8.49c-5.09,0-8.84-2.26-11.25-6.79l-12.52-23.13v74.69c0,5.09-2.55,7.64-7.64,7.64h-31.62c-5.09,0-7.64-2.55-7.64-7.64V112.72c0-5.09,2.55-7.64,7.64-7.64h34.16c4.24,0,7.5,2.12,9.76,6.37l19.1,34.59"
                  />
                  <path
                    class="cls-2"
                    d="M490.66,257.86c-5.09,0-7.64-2.55-7.64-7.64v-25.25c0-5.09,2.55-7.64,7.64-7.64h21.01v-71.3h-21.01c-5.09,0-7.64-2.55-7.64-7.64v-25.68c0-5.09,2.55-7.64,7.64-7.64h90.4c5.09,0,7.64,2.55,7.64,7.64v25.68c0,5.09-2.55,7.64-7.64,7.64h-20.58v71.3h20.58c5.09,0,7.64,2.55,7.64,7.64v25.25c0,5.09-2.55,7.64-7.64,7.64h-90.4Z"
                  />
                </g>
                <g>
                  <path
                    bind:this={dotRef}
                    class="cls-1 theme-dot"
                    d="M206.7,50.26c8.44-3.54,18.13.5,21.63,9.03,3.5,8.53-.49,18.32-8.93,21.87-8.44,3.54-18.12-.5-21.63-9.03-3.51-8.53.49-18.32,8.93-21.87"
                    role="button"
                    tabindex="0"
                    onclick={handleDotClick}
                    onkeydown={(e) => { if (e.key === 'Enter' || e.key === ' ') handleDotClick(e as unknown as MouseEvent); }}
                  />
                  <path
                    class="cls-3"
                    d="M212.43,0h0c2.6,0,4.72,2.14,4.72,4.77v28.69c0,2.62-2.13,4.77-4.72,4.77h0c-2.6,0-4.72-2.15-4.72-4.77V4.77c0-2.63,2.12-4.77,4.72-4.77"
                  />
                  <path
                    class="cls-3"
                    d="M258.49,19.28h0c1.84,1.87,1.84,4.91,0,6.77l-20.06,20.28c-1.84,1.85-4.84,1.85-6.68,0h0c-1.84-1.86-1.84-4.9,0-6.77l20.06-20.28c1.84-1.85,4.85-1.85,6.68,0"
                  />
                  <path
                    class="cls-3"
                    d="M277.57,65.86h0c0,2.64-2.13,4.79-4.72,4.79h-28.37c-2.6,0-4.73-2.15-4.73-4.78h0c0-2.63,2.13-4.78,4.73-4.78h28.37c2.59,0,4.72,2.15,4.72,4.77"
                  />
                  <path
                    class="cls-3"
                    d="M147.29,65.86h0c0-2.63,2.13-4.78,4.72-4.78h28.37c2.6,0,4.72,2.15,4.72,4.78h0c0,2.63-2.12,4.78-4.72,4.78h-28.37c-2.6,0-4.72-2.15-4.72-4.78"
                  />
                  <path
                    class="cls-3"
                    d="M166.37,19.28h0c1.83-1.85,4.84-1.85,6.68,0l20.06,20.28c1.84,1.86,1.84,4.9,0,6.76h0c-1.84,1.86-4.84,1.86-6.68,0l-20.06-20.28c-1.84-1.86-1.84-4.9,0-6.77"
                  />
                </g>
              </g>
              <text class="cls-4" transform="translate(0 326.66)"
                ><tspan x="0" y="0">IT &amp; </tspan><tspan
                  class="cls-5"
                  x="86.8"
                  y="0">A</tspan
                ><tspan class="cls-6" x="116.51" y="0">C</tspan><tspan
                  x="150.37"
                  y="0">ADEMIC SOLUTIONS</tspan
                ></text
              >
            </g>
          </g>
        </svg>
      </div>

      <p class="hero-description animate-fade-in-up-delay-2">
        Transforming complex data into intuitive and actionable insights.
        <br />
        Specialising in Academic Software and Educational Technology.
      </p>
    </div>
  </main>
</div>

<style>
  /* --- Global Resets for `#app` to ensure full screen coverage --- */
  :global(#app) {
    width: 100% !important;
    max-width: 100% !important;
    margin: 0 !important;
    border-inline: none !important;
    padding: 0 !important;
  }

  :global(body) {
    background-color: #1C2329 !important;
    margin: 0 !important;
    padding: 0 !important;
    overflow: hidden !important;
  }

  /* --- Circular Theme Transition Overlay --- */
  .theme-transition-overlay {
    position: fixed;
    inset: 0;
    z-index: 9999;
    pointer-events: none;
    display: none;
    clip-path: circle(0px at 0px 0px);
  }

  /* --- Main Viewport Layout Container --- */
  .app-container {
    position: relative;
    width: 100vw;
    height: 100vh;
    height: 100svh;
    background-color: #1C2329;
    overflow: hidden;
    color: #fff;
    font-family:
      "Google Sans",
      system-ui,
      -apple-system,
      BlinkMacSystemFont,
      "Segoe UI",
      Roboto,
      sans-serif;
    box-sizing: border-box;
  }

  .app-container.light-mode {
    background-color: #f0f2f5;
    color: #1a1a2e;
  }

  .app-container ::selection {
    background-color: #4285f4;
    color: #fff;
  }

  /* --- Interactive Canvas Styles --- */
  .canvas-container {
    position: absolute;
    inset: 0;
    z-index: 0;
    overflow: hidden;
    background-color: #1C2329;
    cursor: crosshair;
  }

  .canvas-container.light-mode {
    background-color: #f0f2f5;
  }

  /* --- Light-mode SVG logo letter overrides --- */
  .light-mode :global(.cls-2) {
    fill: #1C2329;
  }

  .light-mode :global(.cls-4) {
    fill: #1C2329;
  }

  canvas {
    display: block;
    width: 100%;
    height: 100%;
  }

  /* --- Hero Content styling --- */
  .hero-content {
    position: absolute;
    inset: 0;
    z-index: 10;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    pointer-events: none;
    padding: 0 1.5rem;
    box-sizing: border-box;
  }

  .hero-inner {
    max-width: 56rem; /* 4xl */
    width: 100%;
    text-align: center;
    display: flex;
    flex-direction: column;
    gap: 1.75rem;
  }

  .tag-container {
    display: inline-block;
    align-self: center;
    margin-bottom: 50px;
  }

  .hero-tag {
    padding: 0.35rem 0.85rem;
    border: 1px solid rgba(255, 255, 255, 0.15);
    border-radius: 9999px;
    font-size: 0.7rem;
    font-family: ui-monospace, SFMono-Regular, Consolas, monospace;
    color: rgba(255, 255, 255, 0.6);
    letter-spacing: 0.15em;
    text-transform: uppercase;
    background-color: rgba(255, 255, 255, 0.04);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    user-select: none;
    transition: color 0.5s, border-color 0.5s, background-color 0.5s;
  }

  .light-mode .hero-tag {
    border-color: rgba(0, 0, 0, 0.12);
    color: rgba(0, 0, 0, 0.5);
    background-color: rgba(0, 0, 0, 0.04);
  }

  .logo-container {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 100%;
    user-select: none;
    margin: 0;
  }

  .hero-logo {
    display: block;
    height: 250px;
    mix-blend-mode: difference;
    filter: drop-shadow(0 4px 20px rgba(160, 218, 222, 0.15));
    transition: filter 0.5s;
  }

  .light-mode .hero-logo {
    mix-blend-mode: normal;
    filter: drop-shadow(0 4px 16px rgba(37, 190, 194, 0.2));
  }

  /* --- Theme Dot (clickable toggle) --- */
  .theme-dot {
    cursor: pointer;
    pointer-events: auto;
    transition: fill 0.3s, filter 0.3s;
  }

  .theme-dot:hover {
    fill: #fbbf24;
    filter: drop-shadow(0 0 8px rgba(251, 191, 36, 0.6));
  }

  .theme-dot:active {
    fill: #f59e0b;
  }

  @media (min-width: 768px) {
    .hero-logo {
      max-width: 460px;
    }
  }

  @media (min-width: 1024px) {
    .hero-logo {
      max-width: 520px;
    }
  }

  .hero-description {
    max-width: 36rem;
    margin: 0.5rem auto 0;
    font-size: 1rem;
    color: rgba(255, 255, 255, 0.55);
    font-weight: 300;
    line-height: 1.6;
    text-shadow: 0 2px 10px rgba(0, 0, 0, 0.5);
    transition: color 0.5s, text-shadow 0.5s;
  }

  .light-mode .hero-description {
    color: rgba(0, 0, 0, 0.5);
    text-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
  }

  @media (min-width: 768px) {
    .hero-description {
      font-size: 1.15rem;
      max-width: 42rem;
    }
  }

  .animate-fade-in-up {
    animation: fadeInUp 1.2s cubic-bezier(0.16, 1, 0.3, 1) forwards;
  }

  .animate-fade-in-up-delay-1 {
    opacity: 0;
    animation: fadeInUp 1.2s cubic-bezier(0.16, 1, 0.3, 1) 0.15s forwards;
  }

  .animate-fade-in-up-delay-2 {
    opacity: 0;
    animation: fadeInUp 1.2s cubic-bezier(0.16, 1, 0.3, 1) 0.3s forwards;
  }

  @keyframes fadeInUp {
    from {
      opacity: 0;
      transform: translateY(24px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
</style>
