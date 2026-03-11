# Inside JPEG — Product & Technical Specification

**Document version:** 2.0  
**Date:** March 2026  
**Status:** Draft  
**Changelog v2.0:** Scroll direction finalized (vertical); 3D animation system added; inter-section scroll-connected transitions defined; persistent Data Size Bar introduced; collapsible sidebar navigation added; mobile horizontal-within-section pattern formalized.

---

## 1. Overview

### 1.1 Product Summary
"Inside JPEG" is a single-page, scroll-driven, interactive educational web experience that teaches non-technical users how a JPEG image is created — from the moment light enters a camera lens, through all stages of internal camera processing, to every phase of JPEG compression. All interactive demonstrations are personalized by the user's own uploaded photo. The page feels like one continuous, living animation — not a collection of separate sections.

### 1.2 Target Audience
- General public with no background in photography, computer vision, or digital signal processing
- Curious learners, students, hobbyist photographers
- Reading level: accessible, plain English — no assumed prior knowledge

### 1.3 Goals
- Explain the full JPEG pipeline end-to-end, accurately and in depth
- Make every abstract concept tangible through live, image-personalized visualization
- Create a seamless cinematic scroll experience where each section flows into the next
- Use 3D animation and depth wherever it enhances understanding
- Maintain a persistent "data size" indicator so the compression story is always visible
- Be fully self-contained — no backend, no external APIs, no login

---

## 2. Design System

### 2.1 Visual Theme
**Clean & Minimal — Modern Light**

| Token | Value |
|---|---|
| Background primary | `#ffffff` |
| Background secondary | `#f8f9fb` |
| Background accent | `#f0f4ff` |
| Text primary | `#0f172a` |
| Text secondary | `#64748b` |
| Text muted | `#94a3b8` |
| Accent blue | `#2563eb` |
| Accent blue dark | `#1d4ed8` |
| Accent blue tint | `#dbeafe` |
| Border | `#e2e8f0` |
| Border radius | `14px` |
| Card shadow | `0 1px 3px rgba(0,0,0,.06), 0 8px 32px rgba(0,0,0,.07)` |
| 3D scene background | `radial-gradient(#f0f4ff, #ffffff)` |

### 2.2 Typography

| Role | Font | Weight | Size |
|---|---|---|---|
| Page title | Inter | 900 | clamp(2.6rem, 7vw, 5.2rem) |
| Section heading | Inter | 800 | clamp(1.7rem, 3.5vw, 2.6rem) |
| Card heading | Inter | 700 | 1.15rem |
| Body text | Inter | 400 | 1rem |
| Code / matrices | JetBrains Mono | 400/600 | 0.82em |

### 2.3 Layout & Scroll Direction

**Primary scroll direction: Vertical.**

Rationale:
- The JPEG pipeline is a linear narrative — vertical scroll maps naturally to "progress through a story"
- Text + demo stacking (explanation above, visualization below) reads top-to-bottom
- Mobile devices scroll vertically by default; fighting this creates friction for the target audience
- The interactive demos (matrices, canvases, comparison sliders) benefit from vertical real estate

**Horizontal scroll within sections (mobile only):**  
Multi-panel layouts (RGB channels, YCbCr channels, summary cards) that would stack too tightly on narrow screens use a horizontally scrollable row with `scroll-snap-type: x mandatory`. Each panel snaps into place. A subtle scroll-indicator dot row shows position. This is never used for the primary page navigation — only for content carousels inside a section.

**Layout specs:**
- Max content width: `960px`, centered, `24px` horizontal padding
- Section vertical padding: `96px`
- Alternating section backgrounds: white / `#f8f9fb`
- All interactive demo cards: white background, `1px` border, `14px` radius, shadow

### 2.4 Animation Principles

#### General
- All scroll-triggered reveals: `opacity 0→1`, `translateY 28px→0`, duration `0.65s ease`
- Hover states: subtle `translateY(-1px)` lift on primary buttons
- All canvas animations run via `requestAnimationFrame` (60fps target)
- Transitions on data state changes: `0.25–0.4s` easing
- Animations pause when `document.visibilityState === 'hidden'`

#### 3D Animation System
- **Renderer:** Three.js (r128) for all 3D scenes; CSS `perspective` + `transform: rotateX/Y/Z` for simpler 3D card effects
- **Scene defaults:** Soft directional light from top-left; ambient fill light; subtle environment fog on deeper scenes
- **Camera:** Perspective camera, FOV 45°; OrbitControls disabled by default (scroll-driven camera only); mouse parallax on hover adds ±5° tilt
- **Materials:** `MeshPhysicalMaterial` for glass/lens; `MeshStandardMaterial` for solid objects; custom GLSL shader for light ray glow and photon particles
- **3D is used where it enhances spatial understanding** — lens optics, sensor depth, image-as-object, frequency space. It is never decorative noise.

#### Scroll-Connected Inter-Section Transitions
The page uses a **ScrollTimeline** (or `IntersectionObserver` + scroll position fallback) to drive animations based on scroll progress. Each section pair shares a "handoff animation" — a visual element that morphs, moves, or evolves as the user scrolls from one section into the next. See Section 4 for per-transition specifications.

---

## 3. Page Architecture

### 3.1 Section Structure

| # | Section ID | Title | Type |
|---|---|---|---|
| — | `hero` | Landing / Upload | Upload + 3D animated hero |
| 01 | `s-lens` | Light Enters the Lens | 3D scene + particle animation |
| 02 | `s-bayer` | The Image Sensor & Bayer Mosaic | 3D grid + interactive |
| 03 | `s-raw` | Raw Pixel Data | Zoomable pixel inspector |
| 04 | `s-rgb` | Red, Green & Blue Channels | Channel decomposition |
| 05 | `s-ycbcr` | RGB → YCbCr Color Space | Channel decomposition |
| 06 | `s-chroma` | Chroma Subsampling 4:2:0 | Drag-divider comparison |
| 07 | `s-blocks` | 8×8 Block Division | Clickable block selector |
| 08 | `s-dct` | Discrete Cosine Transform | Animated 3D frequency transform |
| 09 | `s-quant` | Quantization | Live quality slider + matrix |
| 10 | `s-zigzag` | Zigzag Scanning | Animated scan sequence |
| 11 | `s-rle` | Run-Length Encoding | Animated token sequence |
| 12 | `s-huffman` | Huffman Coding | 3D tree visualization + code table |
| 13 | `s-quality` | Live JPEG Compression | Full-image quality slider |
| 14 | `s-compare` | Original vs Compressed | Drag-divider comparison |
| 15 | `s-undo` | Undo JPEG — Decode Pipeline | Step-through reconstruction |
| — | `s-summary` | Summary | Pipeline overview cards |

### 3.2 Persistent State Object
A global `S` object holds all shared application state:

```
S {
  img           ImageData      // Uploaded image (max 480px, padded to 8×8)
  w, h          Number         // Image dimensions
  origCanvas    HTMLCanvasElement
  selBx, selBy  Number         // Currently selected 8×8 block origin (pixels)
  block         Object         // { y[], cb[], cr[] } — block channel arrays
  dctC          Float32Array   // DCT coefficients for selected block
  quantC        Array          // Quantized coefficients
  zzSeq         Array          // Zigzag-ordered quantized coefficients
  rleSeq        Array          // Zigzag sequence (input to RLE)
  quality       Number         // Current quality setting (1–100)
  scrollY       Number         // Current window.scrollY, updated on rAF
  sectionProgress Object       // { sectionId: 0–1 } scroll progress per section
}
```

---

## 4. Scroll-Connected Inter-Section Transitions

This is the core of the "one smooth animation" feel. Each pair of adjacent sections shares a visual handoff. The element that ends one section becomes (or morphs into) the element that begins the next.

### Transition Map

| From → To | Handoff Animation |
|---|---|
| **Hero → S01 (Lens)** | The uploaded photo, displayed as a flat card, **3D-rotates on the Y-axis** and flies toward a camera lens model. As it recedes, light rays emanate from its surface toward the lens. The lens scene of S01 builds from this incoming image. |
| **S01 (Lens) → S02 (Bayer)** | The sensor plane at the end of the lens animation **zooms in** until individual photosites are visible. The Bayer color filter array tiles onto the sensor surface in 3D, filters dropping into place one row at a time. |
| **S02 (Bayer) → S03 (Raw Pixels)** | The Bayer grid **flattens** (perspective collapses to orthographic) and the mosaic pattern dissolves into full-color pixels — visualizing demosaicing. The zoomed pixel grid of S03 expands from the center of the Bayer grid. |
| **S03 (Raw) → S04 (RGB)** | The pixel grid **splits apart** into three layers — the R, G, B channels peel off like pages of a book, each sliding left/right/forward in 3D to become the three channel canvases of S04. |
| **S04 (RGB) → S05 (YCbCr)** | The three RGB channel cards **rotate and recolor** — R becomes the warm Cr heatmap, G becomes the Y grayscale, B becomes the cool Cb heatmap — morphing in place with a smooth color transition. |
| **S05 (YCbCr) → S06 (Chroma)** | The Cb and Cr channel canvases **visibly shrink** (scale down to 50% in X and Y), then scale back up with blurred interpolation — demonstrating subsampling visually before S06's demo begins. |
| **S06 (Chroma) → S07 (Blocks)** | The comparison image **overlays a growing grid** — lines sweep across the image left-to-right, top-to-bottom, tiling it into 8×8 blocks. The grid locks into place as S07 becomes active. |
| **S07 (Blocks) → S08 (DCT)** | The **selected block lifts off** the image in 3D (translates forward on Z-axis), rotates slightly, and expands into the DCT demo panel. A spotlight follows it. |
| **S08 (DCT) → S09 (Quant)** | The DCT coefficient matrix **slides left**, a second matrix (the quantization table) **slides in from the right**, and a visual division operation animates between them — coefficients shrinking as the two matrices combine. |
| **S09 (Quant) → S10 (Zigzag)** | The quantized matrix **rotates to face the viewer at a slight angle**, then a glowing cursor traces the zigzag path across it, and the 1D sequence spills out from the bottom-right corner. |
| **S10 (Zigzag) → S11 (RLE)** | The 1D sequence tokens **compress together** — runs of zeros visibly collapse into single pair tokens with a spring animation. The compressed sequence slides into place in S11. |
| **S11 (RLE) → S12 (Huffman)** | The RLE tokens each **sprout a branch** — growing upward into the Huffman tree. The tree builds root-up as the section transitions in. |
| **S12 (Huffman) → S13 (Quality)** | The Huffman tree **folds into a single file icon** that shrinks dramatically (visualizing the final compressed JPEG), then expands back into the full-quality image in S13. |
| **S13 (Quality) → S14 (Compare)** | The quality-adjusted image **splits down the middle** — the left half stays compressed, the right half "heals" back to the original — transitioning directly into the comparison divider of S14. |
| **S14 (Compare) → S15 (Undo)** | The divider **rewinds** (moves from right to left), the compression artifacts visibly dissolve, and the image reconstructs itself — framing the decode narrative of S15. |

### Scroll Progress Binding
Each transition is driven by `scrollProgress` — a normalized 0→1 value representing how far the user has scrolled through the inter-section overlap zone (typically the bottom 30% of one section to the top 30% of the next). Animations are bound to this value via a `lerp()` function so they track scroll exactly, with no fixed-time playback.

```
// Pseudo-code
onScroll(() => {
  const p = getTransitionProgress('s-lens', 's-bayer'); // 0–1
  bayerGrid.scale.set(lerp(0, 1, easeOutCubic(p)));
  lensScene.opacity = lerp(1, 0, p);
});
```

---

## 5. Persistent Data Size Bar

### 5.1 Purpose
A fixed UI element that lives on-screen throughout the entire scrolling journey, showing the **current "virtual file size"** of the image at each stage of the pipeline. This makes the compression story tangible and emotionally satisfying — the user watches the file shrink as they scroll deeper.

### 5.2 Visual Design
- **Position:** Fixed, bottom of viewport, full width, `z-index: 1000`
- **Height:** `52px` (collapsed) / expands to `72px` on hover
- **Background:** `rgba(255,255,255,0.85)`, `backdrop-filter: blur(12px)`, top border `1px solid var(--border)`
- **Contents (left to right):**
  - Stage label: e.g., `"Raw pixels"` → `"After YCbCr"` → `"After Quantization"` (updates per section)
  - Animated fill bar: color transitions from green (large/uncompressed) → yellow → red (tiny/compressed) → green again (decoded)
  - Numeric label: e.g., `"36.0 MB"` → `"12.0 MB"` → `"245 KB"` — animates with a counter roll effect
  - Compression ratio badge: e.g., `"147× smaller"` appears at max compression

### 5.3 Size Values Per Section

The bar shows **representative values** for a typical 12MP photo. When the user uploads an image, values are scaled proportionally based on actual pixel count.

| Section | Stage Label | Approx. File Size | Bar Color |
|---|---|---|---|
| Hero | Raw sensor data | ~36 MB | 🟢 Green |
| S01 | After demosaicing | ~36 MB | 🟢 Green |
| S02–S03 | Full RGB pixels | ~36 MB | 🟢 Green |
| S04 | RGB channels | ~36 MB | 🟢 Green |
| S05 | After YCbCr conversion | ~36 MB | 🟢 Green (no size change — conceptual) |
| S06 | After 4:2:0 subsampling | ~18 MB | 🟡 Yellow (50% chroma reduction) |
| S07 | Divided into blocks | ~18 MB | 🟡 Yellow |
| S08 | After DCT | ~18 MB | 🟡 Yellow (transform only, no size change yet) |
| S09 | After quantization (Q=75) | ~2–4 MB | 🟠 Orange (most zeros now) |
| S10 | After zigzag scan | ~2–4 MB | 🟠 Orange |
| S11 | After RLE | ~800 KB | 🔴 Red (zero runs compressed) |
| S12 | After Huffman coding | ~300 KB | 🔴 Red (final JPEG size) |
| S13 | Live (quality-adjusted) | Dynamic | Dynamic (tracks slider) |
| S14 | Compressed vs. original | Dynamic | Dual indicator |
| S15 | Decoding (rebuilding) | Grows back → 36 MB | 🟢 Returns to green |
| Summary | Final JPEG on disk | ~300 KB | 🔴 Red |

### 5.4 Animation Behavior
- Size value animates with a **rolling counter** (digit-by-digit scroll, duration 600ms) on section change
- Fill bar width transitions with `cubic-bezier(0.4, 0, 0.2, 1)` over 800ms
- Color transitions smoothly through the green→yellow→orange→red spectrum
- On S15 (Undo), the bar **reverses** — animating from red back to green as each decode step is applied, creating a satisfying "uncompressing" effect
- The bar subtly pulses (scale 1.0 → 1.02 → 1.0) when a significant size change occurs

### 5.5 Interaction
- Clicking the bar **expands a tooltip** showing a breakdown: raw size, after each step, and final size
- On mobile, the bar is height `40px` with abbreviated labels

---

## 6. Sidebar Navigation

### 6.1 Purpose
Allows users who have scrolled deep, or who are revisiting, to jump directly to any pipeline section without scrolling the entire page.

### 6.2 Visual Design
- **Position:** Fixed, right edge of viewport, vertically centered, `z-index: 900`
- **Default state:** **Collapsed** — only a small toggle tab is visible (`32×80px`, labeled `"≡ Sections"` rotated 90°, accent blue background)
- **Expanded state:** A `220px` wide panel slides in from the right, `backdrop-filter: blur(16px)`, `background: rgba(255,255,255,0.92)`, left border `1px solid var(--border)`, `border-radius: 14px 0 0 14px`
- **Toggle:** Clicking the tab or pressing `N` on keyboard toggles open/close
- **Close:** Clicking outside the panel, pressing `Escape`, or pressing `N` again

### 6.3 Navigation Panel Contents

```
┌─────────────────────┐
│  § Sections      ×  │  ← header + close button
├─────────────────────┤
│  ○ Upload           │  ← dot = not yet reached
│  ● 01 Light & Lens  │  ← filled = currently in view
│  ✓ 00 Upload        │  ← checkmark = visited/passed
│  ...                │
│  02 Bayer Sensor    │
│  03 Raw Pixels      │
│  04 RGB Channels    │
│  05 YCbCr Space     │
│  06 Chroma Sub.     │
│  07 8×8 Blocks      │
│  08 DCT Transform   │
│  09 Quantization    │
│  10 Zigzag Scan     │
│  11 Run-Length Enc. │
│  12 Huffman Coding  │
│  13 Live Quality    │
│  14 Comparison      │
│  15 Undo JPEG       │
│  ★ Summary          │
└─────────────────────┘
```

### 6.4 Behavior
- **Active section** is highlighted with accent blue background + white text; updated on scroll via `IntersectionObserver`
- **Visited sections** show a small checkmark icon; unvisited show an empty dot
- **Clicking any item** smooth-scrolls to that section and closes the panel
- **Progress indicator:** A thin vertical line on the left edge of the panel acts as a mini scroll-progress bar — the filled portion grows as the user scrolls down
- **On mobile:** The panel is full-screen width when open (`100vw`), with larger tap targets (`48px` row height)
- **Keyboard navigation:** Arrow keys move focus between items when panel is open; Enter jumps to section

### 6.5 Section Labels in Sidebar

| Entry | Short Label | Icon |
|---|---|---|
| Hero | Upload Your Photo | 📷 |
| S01 | Light & Lens | 🔆 |
| S02 | Bayer Sensor | 🟩 |
| S03 | Raw Pixels | 🔍 |
| S04 | RGB Channels | 🎨 |
| S05 | YCbCr Space | 🌈 |
| S06 | Chroma Subsampling | ⬇️ |
| S07 | 8×8 Blocks | 🧩 |
| S08 | DCT Transform | 〰️ |
| S09 | Quantization | ✂️ |
| S10 | Zigzag Scan | ↗️ |
| S11 | Run-Length Enc. | 🗜️ |
| S12 | Huffman Coding | 🌳 |
| S13 | Live Quality | 🎛️ |
| S14 | Comparison | ⟺ |
| S15 | Undo JPEG | 🔄 |
| Summary | Full Pipeline | ★ |

---

## 7. Section Specifications

### HERO — Upload Gate

**Purpose:** First impression + image ingestion.

**3D Scene:**
- A floating camera body renders in Three.js, centered on screen, with a slow idle rotation (Y-axis, ±8°, driven by mouse parallax)
- Particle network background (same as v1.0)
- On upload success: the uploaded photo **appears as a flat 3D card** in front of the camera, facing the user. It rotates 360° on Y-axis (1.5s), then tilts slightly and begins flying toward the lens — this is the **Hero → S01 handoff animation** (see Section 4)

**Behavior:**
- Upload zone: click / drag-drop, any `image/*`
- Image scaled to fit `480×480px`, padded to nearest 8×8
- All demos initialize on upload

---

### SECTION 01 — Light Enters the Lens

**3D Scene (Three.js):**
- Camera lens rendered as a glass cylinder with `MeshPhysicalMaterial` (transmission, roughness ~0.05, IOR 1.5)
- Light rays: `THREE.Line` objects with a glow shader, emanating from the uploaded image card (carrying the handoff from Hero)
- Photon particles: `THREE.Points` with custom shader — particles flow along ray paths, slow near the lens, converge at focal point, then diverge to the sensor plane
- Sensor plane visible at the back: a grid of glowing dots that light up as photons arrive
- The entire 3D scene is scroll-driven — camera slowly orbits as the user scrolls through the section

**Text topics:** Photons; refraction; focal point; lens as funnel.

---

### SECTION 02 — The Image Sensor & Bayer Mosaic

**3D Enhancement:**
- The sensor arrives from the S01 transition already in 3D
- Individual photosites rendered as small 3D cubes with slight depth (extruded squares), colored by their Bayer filter
- On click: the selected photosite **rises up** slightly (Z-translate) and a spotlight illuminates it
- Camera tilts slightly on scroll to show the depth of the photosite wells

**Demo:** Same interactive Bayer grid as v1.0, with 3D depth treatment above.

**Text topics:** Photosites; Bayer mosaic; RGGB; green doubled; colorblind sensor.

---

### SECTION 03 — Raw Pixel Data

**3D Enhancement:**
- The demosaicing visualization from the S02→S03 transition plays (mosaic → full color)
- Zoomed pixel grid: each pixel cell rendered with subtle `rotateX(8deg)` perspective to give a "looking down into the grid" feel
- Pixel cells rise up slightly when hovered

**Demo:** Dual-panel pixel zoom inspector (same as v1.0).

**Text topics:** Demosaicing; pixel = 3 bytes; 36MB uncompressed.

---

### SECTION 04 — RGB Color Channels

**3D Enhancement:**
- Channel separation uses the S03→S04 book-page 3D split transition
- The three channel cards float at slightly different Z-depths with a subtle parallax on scroll
- Mouse hover on any channel card: that card tilts toward the user (CSS `perspective` + `rotateX/Y`)

**Demo:** 4-panel channel display (R, G, B, combined) — horizontally scrollable on mobile.

**Text topics:** Channel separation; green dominance; recombination.

---

### SECTION 05 — YCbCr Color Space

**3D Enhancement:**
- S04→S05 transition: RGB cards rotate and recolor into YCbCr cards in 3D space
- Y channel card is slightly larger and closer (emphasizing its importance)

**Conversion formulas displayed:**
```
Y  =  0.299R + 0.587G + 0.114B
Cb = −0.168736R − 0.331264G + 0.5B + 128
Cr =  0.5R − 0.418688G − 0.081312B + 128
```

**Text topics:** YCbCr; luma/chroma; perceptual asymmetry.

---

### SECTION 06 — Chroma Subsampling (4:2:0)

**3D Enhancement:**
- S05→S06 transition: Cb and Cr cards visibly shrink
- The comparison divider has a subtle 3D shadow/depth effect

**Demo:** Drag-divider comparison + quality slider.

**Text topics:** 4:2:0 meaning; 75% chroma reduction; upsampling on decode.

---

### SECTION 07 — 8×8 Block Division

**3D Enhancement:**
- S06→S07 transition: grid lines sweep across the image
- Selected block lifts off in 3D (prep for S07→S08 transition)
- Grid overlay lines rendered with a glowing blue shader

**Demo:** Clickable block selector; block detail panel. Spacebar = random block.

**Text topics:** Why 8×8; 32,400 blocks in 1080p; independent compression.

---

### SECTION 08 — Discrete Cosine Transform (DCT)

**3D Enhancement:**
- S07→S08 transition: the selected block flies in and expands into the demo
- DCT coefficient grid rendered in **3D as a bar chart** (each of the 64 coefficients is a 3D bar whose height = |value|, color = sign positive/negative). The chart rotates slowly on idle.
- "Animate Transform" button triggers a 3D morph: pixel bars smoothly transition to frequency bars
- The 3D chart can be rotated by dragging

**Demo:** Pixel matrix + 3D DCT bar chart + animate button.

**Text topics:** Frequency domain; DC/AC; energy compaction; equalizer analogy.

---

### SECTION 09 — Quantization

**3D Enhancement:**
- S08→S09 transition: DCT matrix slides left, quantization table slides in
- Quantization shown as a 3D "press" — an overhead plane descends onto the coefficient bar chart, flattening (zeroing) the shorter bars. The press depth is controlled by the quality slider.
- Zeroed bars visually **disappear** (scale to 0 on Y) with a spring animation

**Demo:** Dual-panel matrix + quality slider (shared with S13).

**Quantization formula:**
```
scale = (quality < 50) ? 5000/quality : 200 − quality×2
qt[i] = clamp(round((base[i] × scale + 50) / 100), 1, 255)
```

**Text topics:** Division + rounding = permanent loss; quality as table scalar; zeros = compressibility.

---

### SECTION 10 — Zigzag Scanning

**3D Enhancement:**
- S09→S10 transition: quantized matrix rotates to face viewer at angle
- The zigzag path is rendered as a **glowing 3D wire** tracing across the slightly tilted matrix surface
- Values "pop off" the matrix as tokens that fall into the 1D sequence below

**Demo:** Animated zigzag scan with play button.

**Text topics:** Why diagonal; zero clustering; preparation for RLE.

---

### SECTION 11 — Run-Length Encoding (RLE)

**3D Enhancement:**
- S10→S11 transition: zero tokens visibly compress/collapse
- Token cards have subtle 3D depth; zero-run collapses use a spring-physics crush animation

**Demo:** Before/after token sequence with animated RLE encoding.

**Text topics:** Lossless; pair notation; EOB marker.

---

### SECTION 12 — Huffman Coding

**3D Enhancement:**
- S11→S12 transition: RLE tokens sprout upward into Huffman tree branches
- The Huffman tree is rendered in **3D** (Three.js) as a floating node graph — leaf nodes (symbols) are blue spheres, internal nodes are smaller gray spheres, edges are glowing lines
- Tree auto-rotates slowly; hover over any leaf node shows its symbol, frequency, and binary code in a floating tooltip
- Camera orbits the tree slightly on scroll

**Demo:** 3D Huffman tree + code table grid.

**Text topics:** Variable-length codes; prefix-free; bit savings; used in zip/gzip.

---

### SECTION 13 — Live JPEG Compression

**Demo:** Full-image quality slider (browser native JPEG encoder). File size bar tracks slider.

**Text topics:** Blocky artifacts; banding; ringing; quality as single-value knob.

---

### SECTION 14 — Original vs Compressed

**Demo:** Drag-divider comparison + quality slider.

**Artifact badges:** Blockiness · Color banding · Ringing · Mosquito noise.

**Text topics:** Named artifacts; content sensitivity to compression.

---

### SECTION 15 — Undo JPEG (Decode Pipeline)

**3D Enhancement:**
- S14→S15 transition: divider rewinds, artifacts dissolve
- Each decode step shown on the canvas with a 3D "layer peel" effect — the previous state peels away to reveal the next reconstruction stage underneath
- Data Size Bar reverses (grows) as each decode step is applied

**Demo:** 5-step reconstruction walkthrough.

| Step | Label | Data Size Bar |
|---|---|---|
| 1 | Huffman Decode | Grows slightly |
| 2 | RLE Decode | Grows |
| 3 | Dequantize | Grows |
| 4 | Inverse DCT | Grows more |
| 5 | YCbCr → RGB | Returns to raw size (~36MB) |

**Text topics:** Lossless steps recover perfectly; dequantization cannot recover rounded fractions.

---

### SUMMARY

10-card pipeline overview grid. Horizontally scrollable on mobile.

---

## 8. Technical Requirements

### 8.1 Stack
- **Vanilla HTML5 / CSS3 / JavaScript** — no frameworks, no build step
- Single self-contained `.html` file
- External dependencies: Google Fonts (Inter, JetBrains Mono), Three.js r128 — CDN only
- 2D image processing: native Canvas 2D API
- 3D scenes: Three.js

### 8.2 Rendering Architecture

| Layer | Technology | Responsibility |
|---|---|---|
| 3D scenes | Three.js (WebGL) | Lens, sensor, DCT bar chart, Huffman tree, particle systems |
| 2D image processing | Canvas 2D API | Pixel zoom, channel decomposition, comparisons, matrices |
| CSS 3D | CSS perspective + transforms | Channel card depth, hover tilts, token spring animations |
| DOM animations | CSS transitions + Web Animations API | Scroll reveals, counter rolls, button states |
| Scroll binding | IntersectionObserver + scroll events | Section progress, inter-section transitions, sidebar active state |

### 8.3 Client-Side Algorithm Inventory

| Algorithm | Location |
|---|---|
| Image scaling + 8×8 padding | Upload handler |
| RGB → YCbCr conversion | Sections 05, 06 |
| Bayer pattern simulation | Section 02 |
| 2D DCT (Type-II, 8×8) | Section 08 |
| 2D IDCT (Type-III, 8×8) | Section 15 |
| Standard luminance quantization table | Sections 09, 10, 11 |
| Quality → QT scaling formula | Sections 09, 13 |
| 4:2:0 chroma subsampling + reconstruction | Section 06 |
| Zigzag scan (lookup table) | Sections 10, 11, 12 |
| Run-length encoding | Section 11 |
| Huffman tree construction | Section 12 |
| JPEG encode/decode (native) | Sections 13, 14 |
| Scroll progress computation | Inter-section transitions |
| lerp / easing functions | All scroll-driven animations |

### 8.4 Performance Requirements
- 60fps on modern desktop hardware; 30fps acceptable on mid-range mobile
- Three.js scenes: max 50K triangles per scene; dispose of geometries/materials when section scrolls out of view
- Slider interactions: debounce heavy ops at 16ms
- Initial page load: < 500KB excluding fonts and Three.js (~600KB); lazy-initialize Three.js scenes only when their section is within 1 viewport of the current scroll position
- Block-iterating operations (DCT over full image): use `setTimeout` chunking or Web Worker

### 8.5 Responsive Design
- All layouts adapt via CSS Grid auto-fill and `clamp()`
- Grid columns collapse to 1 on viewport < 680px
- Multi-panel sections use horizontal `scroll-snap` carousels on mobile
- Canvas elements scale via CSS `width: 100%`
- Three.js renderers resize on `window.resize`
- Touch events on drag-dividers and block selectors
- Sidebar: full-screen overlay on mobile

### 8.6 Accessibility
- All interactive elements keyboard-focusable
- Color is never the sole carrier of meaning
- `aria-label` on all canvas elements
- Sidebar panel traps focus when open (`focus-trap`)
- Reduced-motion: `@media (prefers-reduced-motion: reduce)` disables all scroll-driven 3D transitions; sections still reveal with simple fade-in
- Data Size Bar readable by screen reader (`aria-live="polite"` on the numeric label)

### 8.7 Browser Support
- Chrome 100+, Firefox 100+, Safari 15+, Edge 100+
- Requires: WebGL 1.0, Canvas 2D API, FileReader, `toDataURL`, IntersectionObserver, CSS Grid, CSS Custom Properties, `backdrop-filter`

---

## 9. Interactivity Map

| Interaction | Trigger | Effect |
|---|---|---|
| Image upload | Click / drag-drop | Loads image, triggers Hero 3D animation, initializes all demos |
| Bayer cell click | Click on Bayer grid | 3D rise animation, shows pixel metadata |
| Raw pixel click | Click on source image | Draws zoomed pixel grid |
| Block click | Click on image in S07 | Block lifts in 3D, updates S08–S12, triggers S07→S08 handoff |
| Spacebar | Anywhere | Random block selection |
| DCT Animate | Button | 3D pixel bars morph to frequency bars |
| Quality slider | S09 / S13 shared | Updates quantization matrix, 3D quant press, full image, file size, Data Size Bar |
| Zigzag Play | Button | 3D glowing wire traces scan, tokens pop off matrix |
| Apply RLE | Button | Spring-physics token collapse animation |
| Drag divider | S06 / S14 | Comparison overlay slides |
| Undo step buttons | S15 | 3D layer peel + Data Size Bar grows |
| Sidebar toggle | Click tab / press N | Panel slides in/out from right |
| Sidebar item click | Click nav item | Smooth-scrolls to section, closes panel |
| Data Size Bar click | Click bar | Expands size breakdown tooltip |
| Huffman node hover | Hover 3D node | Floating tooltip with symbol, frequency, code |
| DCT chart drag | Drag on 3D chart | Rotates the coefficient bar chart |
| Scroll | Window scroll | Drives all inter-section transitions, updates sidebar active state, updates Data Size Bar |

---

## 10. Content & Tone Guidelines

- **No jargon without immediate explanation.** Every technical term defined on first use.
- **Analogies first, math second.** Everyday analogies before formulas.
- **Math is shown but never required.** Formulas in code blocks for curious readers only.
- **Active, present-tense voice.** "JPEG divides your image…" not "The image is divided…"
- **Second person.** "your photo" throughout.
- **Section text:** 2 short paragraphs maximum; dense detail goes into demo tooltips.

---

## 11. Open Questions / Future Enhancements

| Item | Notes |
|---|---|
| Progressive JPEG mode | Interlaced load order visualization |
| Chrominance quantization table | Currently only luminance shown |
| JFIF / EXIF header bytes | Show the raw file bytes in a hex viewer |
| Export compressed image | Download quality-adjusted JPEG |
| WebWorker for DCT | Offload block iteration for large images |
| Three.js scene sharing | Hero and S01 share one renderer for the handoff — requires careful scene graph design |
| Scroll-linked animation performance | `ScrollTimeline` (CSS) has limited browser support; polyfill or rAF fallback needed |
| Data Size Bar on very small screens | May need to collapse to icon-only below 360px width |