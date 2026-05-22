# Lumi Solutions — Interactive Landing Page

Lumi Solutions is a premium, interactive landing page designed for **Educational Technology Specialists** who specialize in transforming complex data into intuitive, actionable academic insights. It is a modern single page application built with cutting-edge web technologies to deliver a seamless user experience.

---

## 🌟 Key Features

### 1. High-Performance Interactive Physics Engine
* **Elastic Particle Collisions**: Realistic circle-circle elastic bouncing based on particle sizes (acting as masses) and collision normal vectors.
* **Spring Dynamics**: Interactive particles are anchored to floating origins via simulated elastic springs, returning smoothly after mouse displacement.
* **Mouse Repulsion**: Fluid mouse-tracking push force that displaces foreground particles within a custom repulsion field.
* **60 FPS Performance**: Physics computations leverage raw high-frequency JavaScript arrays to sidestep Svelte's reactivity overhead during animation ticks, keeping rendering exceptionally smooth.

### 2. Multi-Layer Parallax Snow Simulation
* **Ambient Background Snow**: Soft, distant snowflakes falling downwards at randomized rates (`0.15` to `0.45` pixels/frame) with a gentle horizontal harmonic sway (`Math.sin`).
* **Interactive Foreground Snow**: Foregrounds act as snow particles that drift down the screen. Larger snowflakes fall naturally faster than smaller ones to evoke realistic 3D depth of field.
* **Robust Boundary Wrapping**: Complete wrap-around coordinates for both falling particles and their anchor points, avoiding sudden slingshots or snapping glitches.

### 3. Premium Aesthetics & Typography
* **Google Sans Typography**: Styled with high-fidelity, professional **Google Sans** typography, loaded directly via Google Fonts preconnected CDN to offer a clean, premium visual aesthetic.
* **Mix-Blend Difference Blending**: The **Lumi Solutions** logo uses a sophisticated CSS `mix-blend-mode: difference` and a soft custom teal glow drop-shadow, allowing falling snow particles to dynamically contrast and blend as they pass behind it.
* **Google Blue Accents**: Subtle hints of branding integrated using vibrant HSL colors and soft radial gradient pulses.

---

## 🛠️ Technical Stack

* **Framework**: Svelte 5 (incorporating modern runes like `$state`, `$effect`, and native event binds)
* **Language**: TypeScript (100% strict type safety check)
* **Build Tool**: Vite
* **Typography**: Google Sans (integrated via preconnected Google Fonts API)
* **Styles**: Native Scoped Vanilla CSS (Zero utility library bloat for maximum loading speed and SEO optimization)

---

## 📐 Physics and Math Core

The engine runs three key mathematical phases on each animation frame inside the Svelte `$effect` lifecycle:

1. **Drift & Sway**:
   $$\text{originY} \leftarrow \text{originY} + (0.12 + \text{size} \times 0.08)$$
   $$\text{originX} \leftarrow \text{originX} + \sin(\text{time} \times 0.001 + \text{angle}) \times 0.08$$
2. **Spring Return (Hooke's Law approximation)**:
   $$v_x \leftarrow v_x + (\text{originX} - x) \times \text{RETURN\_SPEED}$$
   $$v_y \leftarrow v_y + (\text{originY} - y) \times \text{RETURN\_SPEED}$$
3. **Elastic Circle Collision**:
   Resolves overlap statically, then redirects velocities based on mass proportions (represented by particle area/size) and a high coefficient of restitution ($0.85$):
   $$\vec{v}_1' = \vec{v}_1 + \frac{\vec{I}}{m_1}, \quad \vec{v}_2' = \vec{v}_2 - \frac{\vec{I}}{m_2}$$

---

## 📄 License

Created with passion for **Lumi Solutions**.