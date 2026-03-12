# Phase 09 — 3D Scenes & Animations (Three.js)

**Goal:** Add all Three.js 3D scenes and CSS 3D effects to sections that were built with 2D fallbacks in previous phases. This phase transforms the experience from informative to cinematic.

**Depends on:** All previous phases (01–08). Each 3D enhancement replaces or augments an existing 2D demo.

---

## Three.js Setup

### Shared Renderer Architecture
- Create Three.js renderers **lazily** — only when the section is within 1 viewport of scroll position
- Dispose geometries, materials, and textures when the section scrolls out of view (beyond 2 viewports)
- Max 50K triangles per scene
- All 3D canvases: `antialias: true`, `alpha: true` (transparent background)

### Scene Defaults
- **Lighting:** Soft directional light from top-left + ambient fill light
- **Camera:** Perspective camera, FOV 45°
- **OrbitControls:** Disabled by default (scroll-driven camera movement)
- **Mouse parallax:** On hover, camera shifts ±5° tilt based on mouse position
- **Background:** Transparent (CSS `var(--scene-bg)` on the container)

### Reduced Motion
All 3D animations are disabled under `prefers-reduced-motion: reduce`. Show the final state of each 3D scene statically instead.

---

## Hero — 3D Camera Model

**Replaces:** Static hero layout (from Phase 02)

### Scene
- A floating camera body rendered in Three.js, centered on screen
- Model: simplified geometric camera (box body + cylinder lens + small prism viewfinder)
- Material: `MeshStandardMaterial` with metallic gray finish
- Idle animation: slow Y-axis rotation (±8°), driven by mouse parallax
- Particle network background: floating dots connected by thin lines, drifting slowly

### Post-Upload Animation
1. The uploaded photo appears as a **flat 3D card** (`PlaneGeometry` with the image as texture) in front of the camera
2. Card rotates 360° on Y-axis (1.5s, ease-in-out)
3. Card tilts slightly and begins flying toward the camera lens
4. Light rays emanate from the card's surface toward the lens
5. This transitions into S01's lens scene (see Phase 10 handoff)

---

## Section 01 — 3D Lens & Light Rays

**Replaces:** 2D lens diagram (from Phase 04)

### Scene
- **Camera lens:** Glass cylinder using `MeshPhysicalMaterial`:
  - `transmission: 0.9`
  - `roughness: 0.05`
  - `ior: 1.5`
  - `thickness: 0.5`
- **Light rays:** `THREE.Line` objects with custom glow shader (additive blending)
  - Emanate from left side (where uploaded image was)
  - Colors sampled from the uploaded image
  - Converge through the lens to a focal point
  - Diverge from focal point to sensor plane on right
- **Photon particles:** `THREE.Points` with custom shader
  - Small glowing dots flowing along ray paths
  - Slow down near the lens surface
  - Converge at focal point
  - Spread to sensor plane
  - Use `BufferGeometry` with position + velocity attributes
- **Sensor plane:** Grid of small dots at the back, glow up as photons arrive
- **Scroll-driven:** Camera slowly orbits as user scrolls through section

### Custom GLSL (Light Ray Glow)
```glsl
// Vertex shader
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
// Fragment shader
uniform float uTime;
uniform vec3 uColor;
varying vec2 vUv;
void main() {
  float glow = smoothstep(0.5, 0.0, abs(vUv.y - 0.5));
  float pulse = 0.8 + 0.2 * sin(uTime * 3.0 + vUv.x * 10.0);
  gl_FragColor = vec4(uColor * glow * pulse, glow * 0.7);
}
```

---

## Section 02 — 3D Bayer Sensor

**Enhances:** 2D Bayer grid (from Phase 04)

### 3D Treatment
- Individual photosites as small extruded cubes (`BoxGeometry`, slight height)
- Colored by Bayer filter (R: red, G: green, B: blue) using `MeshStandardMaterial`
- Camera tilted ~15° on X-axis to show depth (looking down into the wells)
- On click: selected photosite **rises up** on Z-axis (`translateZ(0.3)`) with spotlight
- Scroll-driven: camera tilt changes slightly as user scrolls

### Integration
- Overlay the 3D scene on top of the 2D interactive grid
- 2D click handling still works underneath
- 3D provides the visual depth; 2D provides the interactivity

---

## Section 03 — 3D Pixel Grid

**Enhances:** 2D pixel inspector (from Phase 04)

### 3D Treatment
- Zoomed pixel grid: each pixel cell has subtle CSS `perspective` + `rotateX(8deg)`
- Creates a "looking down into the grid" feel
- Pixel cells rise slightly on hover (CSS `transform: translateZ(4px)`)
- This is CSS 3D, not Three.js — simpler and more performant

```css
.pixel-grid {
  perspective: 800px;
  transform-style: preserve-3d;
}
.pixel-cell {
  transform: rotateX(8deg);
  transition: transform 0.2s ease;
}
.pixel-cell:hover {
  transform: rotateX(8deg) translateZ(4px);
}
```

---

## Section 04 — 3D Channel Cards

**Enhances:** 2D channel panels (from Phase 05)

### CSS 3D Treatment
- Three channel cards float at slightly different Z-depths
- Subtle parallax on scroll (cards shift in Z based on scroll progress)
- Mouse hover: card tilts toward the user

```css
.channel-card {
  perspective: 1000px;
  transform-style: preserve-3d;
}
.channel-card:hover {
  transform: rotateX(-3deg) rotateY(5deg) translateZ(8px);
  transition: transform 0.3s ease;
}
```

---

## Section 05 — 3D YCbCr Cards

**Enhances:** 2D YCbCr panels (from Phase 05)

### CSS 3D Treatment
- Y channel card is slightly larger and closer (emphasizing importance)
- `transform: scale(1.05) translateZ(16px)` on Y card
- Same hover tilt as S04

---

## Section 08 — 3D DCT Bar Chart

**Replaces:** 2D DCT coefficient matrix (from Phase 06 — matrix remains as secondary view)

### Scene
- 64 bars arranged in an 8×8 grid on a flat plane
- Each bar: `BoxGeometry` with height = `|coefficient value|` (normalized)
- Color: positive values = blue, negative values = red/orange
- DC coefficient (top-left) typically tallest — visually dominant
- Idle: chart rotates slowly on Y-axis (~0.5 RPM)

### Animate Transform
- "Animate Transform" button triggers morph animation:
  - Bars start at heights representing pixel values (all roughly similar height)
  - Morph to heights representing DCT coefficients (few tall, many short/zero)
  - Duration: 1.5s with ease-in-out
  - Use `lerp` to interpolate heights each frame

### Drag to Rotate
- Enable `OrbitControls` on the DCT chart (limited to Y rotation + slight X tilt)
- Touch: pinch/drag to rotate

---

## Section 09 — 3D Quantization Press

**Enhances:** 2D quantization visualization (from Phase 06)

### Scene
- Reuse the DCT bar chart from S08 (or create a copy)
- Add a flat transparent "press" plane above the bars
- Animation: the press descends, flattening bars below a threshold
  - Press depth controlled by quality slider
  - High quality: press barely touches → few bars affected
  - Low quality: press descends deep → most bars crushed to zero
- Zeroed bars: `scaleY(0)` with spring animation (overshoot + settle)
- Materials: press plane uses `MeshPhysicalMaterial` with transmission (glass-like)

---

## Section 12 — 3D Huffman Tree

**Replaces:** 2D tree visualization (from Phase 07)

### Scene
- Floating node graph in 3D space
- Leaf nodes: blue spheres (`SphereGeometry`, `MeshStandardMaterial` accent blue)
- Internal nodes: smaller gray spheres
- Edges: glowing lines connecting nodes (`THREE.Line` with emissive material)
- Layout: recursive 3D positioning — root at top, leaves spread outward and downward
- Auto-rotation: slow Y-axis rotation (~0.3 RPM)
- Camera orbits slightly on scroll

### Hover Interaction
- Raycasting on mouse move to detect hovered nodes
- On leaf hover: floating HTML tooltip (CSS-positioned over the 3D canvas) showing:
  - Symbol value
  - Frequency
  - Binary code
- Highlight the path from root to the hovered leaf (edges glow brighter)

---

## Shared 3D Utilities

### Lazy Initialization
```js
const sceneManagers = {};

function initSceneIfNeeded(sectionId, initFn) {
  if (sceneManagers[sectionId]) return;
  const wrapper = document.querySelector(`#${sectionId} .three-container`);
  if (!wrapper) return;

  const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(wrapper.clientWidth, wrapper.clientHeight);
  wrapper.appendChild(renderer.domElement);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(45, wrapper.clientWidth / wrapper.clientHeight, 0.1, 100);

  // Lighting
  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(-2, 3, 2);
  scene.add(dirLight);
  scene.add(new THREE.AmbientLight(0xffffff, 0.4));

  const manager = { renderer, scene, camera, wrapper };
  initFn(manager);
  sceneManagers[sectionId] = manager;
}
```

### Scene Disposal
```js
function disposeScene(sectionId) {
  const mgr = sceneManagers[sectionId];
  if (!mgr) return;
  mgr.scene.traverse(obj => {
    if (obj.geometry) obj.geometry.dispose();
    if (obj.material) {
      if (Array.isArray(obj.material)) obj.material.forEach(m => m.dispose());
      else obj.material.dispose();
    }
  });
  mgr.renderer.dispose();
  mgr.wrapper.removeChild(mgr.renderer.domElement);
  delete sceneManagers[sectionId];
}
```

### Scroll-Driven Lazy Loading
```js
// In the main scroll loop:
const vh = window.innerHeight;
Object.entries(sectionInitFns).forEach(([id, initFn]) => {
  const el = document.getElementById(id);
  if (!el) return;
  const rect = el.getBoundingClientRect();
  const inRange = rect.top < vh * 2 && rect.bottom > -vh;
  if (inRange) initSceneIfNeeded(id, initFn);
  else if (rect.top > vh * 3 || rect.bottom < -vh * 2) disposeScene(id);
});
```

### Mouse Parallax
```js
function addMouseParallax(camera, basePosition, maxAngle = 5) {
  document.addEventListener('mousemove', e => {
    const x = (e.clientX / window.innerWidth - 0.5) * 2;
    const y = (e.clientY / window.innerHeight - 0.5) * 2;
    camera.rotation.y = basePosition.ry + THREE.MathUtils.degToRad(x * maxAngle);
    camera.rotation.x = basePosition.rx + THREE.MathUtils.degToRad(y * maxAngle);
  });
}
```

### Resize Handler
```js
window.addEventListener('resize', () => {
  Object.values(sceneManagers).forEach(mgr => {
    const w = mgr.wrapper.clientWidth;
    const h = mgr.wrapper.clientHeight;
    mgr.camera.aspect = w / h;
    mgr.camera.updateProjectionMatrix();
    mgr.renderer.setSize(w, h);
  });
});
```

---

## Performance Budget

- Max 50K triangles per active scene
- Only 1–2 scenes active simultaneously (lazy load + dispose)
- Use `BufferGeometry` for all particle systems
- Limit pixel ratio: `Math.min(window.devicePixelRatio, 2)`
- 60fps target on desktop; 30fps acceptable on mobile

---

## Acceptance Criteria

- [ ] Hero: 3D camera model renders and idles with rotation
- [ ] Hero: Post-upload card animation plays (rotate, fly toward lens)
- [ ] S01: 3D lens with glass material renders correctly
- [ ] S01: Light rays with glow shader animate along paths
- [ ] S01: Photon particles flow through the scene
- [ ] S01: Scene is scroll-driven (camera orbits on scroll)
- [ ] S02: Bayer grid has 3D extruded photosites
- [ ] S02: Selected photosite rises with spotlight
- [ ] S03: Pixel grid has CSS 3D perspective effect
- [ ] S04: Channel cards have CSS 3D depth and hover tilt
- [ ] S08: 3D bar chart renders DCT coefficients
- [ ] S08: Animate Transform morphs pixel bars to frequency bars
- [ ] S08: Chart is draggable for rotation
- [ ] S09: 3D press animation controlled by quality slider
- [ ] S09: Zeroed bars disappear with spring animation
- [ ] S12: 3D Huffman tree renders with correct structure
- [ ] S12: Hover shows floating tooltip with symbol info
- [ ] S12: Path highlighting on leaf hover
- [ ] Scenes lazy-load when within 1 viewport
- [ ] Scenes dispose when scrolled far away
- [ ] Mouse parallax works on all 3D scenes
- [ ] Reduced motion: 3D animations disabled, static state shown
- [ ] Performance: maintains 60fps on desktop
- [ ] Resize: all 3D canvases resize correctly
