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
    glowGradient.addColorStop(0, `rgba(66, 133, 244, ${pulseOpacity})`); // Soft Google Blue glow
    glowGradient.addColorStop(1, "rgba(0, 0, 0, 0)");

    ctx.fillStyle = glowGradient;
    ctx.fillRect(0, 0, canvasWidth, canvasHeight);

    // --- Render Part 2: Background Drifting Stars (Snow) ---
    ctx.fillStyle = "#ffffff";
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

      ctx.fillStyle =
        p.color === "#ffffff" ? `rgba(255, 255, 255, ${opacity})` : p.color;

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

<div class="app-container">
  <!-- Interactive Physics Canvas Background -->
  <div
    bind:this={containerRef}
    class="canvas-container"
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
        <img src="/logo.svg" alt="Lumi Solutions Logo" class="hero-logo" />
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
    background-color: #000 !important;
    margin: 0 !important;
    padding: 0 !important;
    overflow: hidden !important;
  }

  /* --- Main Viewport Layout Container --- */
  .app-container {
    position: relative;
    width: 100vw;
    height: 100vh;
    height: 100svh;
    background-color: #000;
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
    background-color: #000;
    cursor: crosshair;
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
    width: 100%;
    max-width: min(100%, 360px);
    height: auto;
    mix-blend-mode: difference;
    filter: drop-shadow(0 4px 20px rgba(160, 218, 222, 0.15));
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
