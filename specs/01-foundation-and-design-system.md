# Phase 01 — Foundation & Design System

**Goal:** Create the single HTML file with the complete design system, page skeleton, global state, and scroll infrastructure. This is the shell that all subsequent phases build into.

---

## Deliverable

A single `index.html` file containing:
1. All CSS custom properties and design tokens
2. Typography system
3. Page skeleton with all section containers (empty)
4. Global state object `S`
5. Scroll observation infrastructure
6. Responsive layout system

---

## 1. File Structure

Single self-contained `index.html` file. All CSS in a `<style>` block. All JS in a `<script>` block at the end of `<body>`.

External dependencies (CDN only):
- Google Fonts: Inter (weights 400, 700, 800, 900), JetBrains Mono (weights 400, 600)
- Three.js r128: `https://unpkg.com/three@0.128.0/build/three.min.js` (load with `defer`)

---

## 2. Design Tokens (CSS Custom Properties)

```css
:root {
  --bg-primary: #ffffff;
  --bg-secondary: #f8f9fb;
  --bg-accent: #f0f4ff;
  --text-primary: #0f172a;
  --text-secondary: #64748b;
  --text-muted: #94a3b8;
  --accent-blue: #2563eb;
  --accent-blue-dark: #1d4ed8;
  --accent-blue-tint: #dbeafe;
  --border: #e2e8f0;
  --radius: 14px;
  --shadow-card: 0 1px 3px rgba(0,0,0,.06), 0 8px 32px rgba(0,0,0,.07);
  --scene-bg: radial-gradient(#f0f4ff, #ffffff);
}
```

---

## 3. Typography

| Role | Font | Weight | Size |
|---|---|---|---|
| Page title | Inter | 900 | clamp(2.6rem, 7vw, 5.2rem) |
| Section heading | Inter | 800 | clamp(1.7rem, 3.5vw, 2.6rem) |
| Card heading | Inter | 700 | 1.15rem |
| Body text | Inter | 400 | 1rem |
| Code / matrices | JetBrains Mono | 400/600 | 0.82em |

Base styles:
- `body`: `font-family: 'Inter', sans-serif; color: var(--text-primary); line-height: 1.6; margin: 0; overflow-x: hidden;`
- `*, *::before, *::after`: `box-sizing: border-box`

---

## 4. Page Skeleton HTML

Create all section containers in order. Each section is a `<section>` element with its `id` attribute. Sections alternate backgrounds between `--bg-primary` and `--bg-secondary`.

```html
<main>
  <section id="hero" class="section"><!-- Phase 02 --></section>
  <section id="s-lens" class="section section--alt"><!-- Phase 04 --></section>
  <section id="s-bayer" class="section"><!-- Phase 04 --></section>
  <section id="s-raw" class="section section--alt"><!-- Phase 04 --></section>
  <section id="s-rgb" class="section"><!-- Phase 05 --></section>
  <section id="s-ycbcr" class="section section--alt"><!-- Phase 05 --></section>
  <section id="s-chroma" class="section"><!-- Phase 05 --></section>
  <section id="s-blocks" class="section section--alt"><!-- Phase 06 --></section>
  <section id="s-dct" class="section"><!-- Phase 06 --></section>
  <section id="s-quant" class="section section--alt"><!-- Phase 06 --></section>
  <section id="s-zigzag" class="section"><!-- Phase 07 --></section>
  <section id="s-rle" class="section section--alt"><!-- Phase 07 --></section>
  <section id="s-huffman" class="section"><!-- Phase 07 --></section>
  <section id="s-quality" class="section section--alt"><!-- Phase 08 --></section>
  <section id="s-compare" class="section"><!-- Phase 08 --></section>
  <section id="s-undo" class="section section--alt"><!-- Phase 08 --></section>
  <section id="s-summary" class="section"><!-- Phase 08 --></section>
</main>
```

Section CSS:
- `.section`: `max-width: 960px; margin: 0 auto; padding: 96px 24px; position: relative;`
- Wrap each section in a full-width div if alternating background is needed (section background should span full viewport width, content centered at 960px)
- `.section--alt` parent wrapper: `background: var(--bg-secondary);`

---

## 5. Global State Object

```js
const S = {
  img: null,           // ImageData — uploaded image (max 480px, padded to 8×8)
  w: 0,                // Image width
  h: 0,                // Image height
  origCanvas: null,    // HTMLCanvasElement — original uploaded image
  selBx: 0,            // Selected 8×8 block origin X (pixels)
  selBy: 0,            // Selected 8×8 block origin Y (pixels)
  block: null,         // { y: [], cb: [], cr: [] } — block channel arrays
  dctC: null,          // Float32Array — DCT coefficients for selected block
  quantC: null,        // Array — quantized coefficients
  zzSeq: null,         // Array — zigzag-ordered quantized coefficients
  rleSeq: null,        // Array — RLE-encoded sequence
  quality: 75,         // Current quality setting (1–100)
  scrollY: 0,          // Current window.scrollY
  sectionProgress: {}, // { sectionId: 0–1 } scroll progress per section
};
```

---

## 6. Scroll Infrastructure

### IntersectionObserver Setup
- Create an observer for each `<section>` to track visibility
- Compute `sectionProgress` (0–1) for each section based on scroll position within that section's bounds
- Update `S.scrollY` on every `requestAnimationFrame`
- Compute transition progress between adjacent sections (bottom 30% of one, top 30% of next)

```js
// Scroll progress computation
function updateScrollProgress() {
  S.scrollY = window.scrollY;
  const vh = window.innerHeight;
  document.querySelectorAll('.section-wrapper').forEach(wrapper => {
    const rect = wrapper.getBoundingClientRect();
    const id = wrapper.querySelector('.section').id;
    const progress = Math.max(0, Math.min(1,
      (vh - rect.top) / (vh + rect.height)
    ));
    S.sectionProgress[id] = progress;
  });
  requestAnimationFrame(updateScrollProgress);
}
requestAnimationFrame(updateScrollProgress);
```

### Scroll-triggered reveal
- Elements with class `.reveal` start with `opacity: 0; transform: translateY(28px);`
- When section progress > threshold, animate to `opacity: 1; transform: translateY(0);` over `0.65s ease`
- Use `IntersectionObserver` for triggering

### Transition progress helper
```js
function getTransitionProgress(fromId, toId) {
  // Returns 0–1 based on scroll in the overlap zone between two sections
}

function lerp(a, b, t) { return a + (a - b) * t; }

function easeOutCubic(t) { return 1 - Math.pow(1 - t, 3); }
```

---

## 7. Animation Pause on Hidden

```js
document.addEventListener('visibilitychange', () => {
  // Set a global flag; all rAF loops check this before running
});
```

---

## 8. Reduced Motion Support

```css
@media (prefers-reduced-motion: reduce) {
  .reveal {
    opacity: 1 !important;
    transform: none !important;
    transition: none !important;
  }
  /* 3D transitions disabled — simple fade-in only */
}
```

---

## 9. Responsive Layout

- Max content width: `960px`, centered
- Horizontal padding: `24px`
- Section vertical padding: `96px`
- Grid columns collapse to 1 on `viewport < 680px`
- All interactive demo cards: white background, `1px solid var(--border)`, `14px` radius, `var(--shadow-card)`

```css
.demo-card {
  background: var(--bg-primary);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  box-shadow: var(--shadow-card);
  padding: 24px;
}
```

---

## 10. Accessibility Base

- `html { scroll-behavior: smooth; }`
- All interactive elements must be keyboard-focusable (ensured in later phases)
- Add `<meta name="viewport" content="width=device-width, initial-scale=1">`
- Add `<meta charset="UTF-8">`
- Add page `<title>Inside JPEG — How JPEG Compression Works</title>`

---

## Acceptance Criteria

- [ ] `index.html` loads in browser with no errors
- [ ] All CSS variables are defined and usable
- [ ] All 17 section containers exist in the DOM with correct IDs
- [ ] Alternating backgrounds visible
- [ ] `S` object is accessible in console
- [ ] Scroll progress updates in real-time (verify via `console.log(S.sectionProgress)`)
- [ ] `.reveal` elements animate on scroll
- [ ] Reduced motion media query disables animations
- [ ] Page is responsive — content centered at 960px, padding on mobile
- [ ] Fonts load correctly (Inter, JetBrains Mono)
