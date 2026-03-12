# Phase 10 — Scroll-Connected Transitions & Final Polish

**Goal:** Implement all inter-section scroll-connected transitions that create the "one continuous animation" feel, plus final polish and performance optimization.

**Depends on:** All previous phases (01–09). This phase ties everything together.

---

## Scroll Transition System

### Architecture
Each transition is driven by a normalized `scrollProgress` value (0→1) representing how far the user has scrolled through the overlap zone between two adjacent sections. The overlap zone is the bottom 30% of one section to the top 30% of the next.

### Progress Binding
```js
function getTransitionProgress(fromId, toId) {
  const fromEl = document.getElementById(fromId)?.closest('.section-wrapper');
  const toEl = document.getElementById(toId)?.closest('.section-wrapper');
  if (!fromEl || !toEl) return 0;

  const fromRect = fromEl.getBoundingClientRect();
  const toRect = toEl.getBoundingClientRect();
  const vh = window.innerHeight;

  // Overlap zone: bottom 30% of "from" to top 30% of "to"
  const overlapStart = fromRect.bottom - fromRect.height * 0.3;
  const overlapEnd = toRect.top + toRect.height * 0.3;

  const progress = 1 - (overlapEnd - vh / 2) / (overlapEnd - overlapStart);
  return Math.max(0, Math.min(1, progress));
}
```

All animations use `lerp()` and easing functions — no fixed-time playback. Animations track scroll position exactly.

---

## Transition Specifications

### 1. Hero → S01 (Lens)
**Handoff:** The uploaded photo (3D card) flies toward a camera lens model.

- Progress 0–0.3: Photo card rotates on Y-axis (0° → 15°)
- Progress 0.3–0.7: Card scales down (1 → 0.3) and translates toward lens
- Progress 0.5–1.0: Light rays begin emanating from card surface, growing in intensity
- Progress 0.8–1.0: Card fades to opacity 0; lens scene fully visible

```js
function transitionHeroToLens(p) {
  const ep = easeOutCubic(p);
  heroCard.rotation.y = lerp(0, Math.PI * 0.08, ep);
  heroCard.scale.setScalar(lerp(1, 0.3, ep));
  heroCard.position.z = lerp(0, -5, ep);
  heroCard.material.opacity = lerp(1, 0, smoothstep(0.7, 1, p));
  lensScene.opacity = lerp(0, 1, smoothstep(0.3, 0.8, p));
}
```

### 2. S01 (Lens) → S02 (Bayer)
**Handoff:** Sensor plane zooms in until photosites are visible.

- Progress 0–0.5: Camera zooms into the sensor plane at the end of the lens scene
- Progress 0.5–0.8: Bayer color filters drop onto the sensor surface (one row at a time, from top)
- Progress 0.8–1.0: Full Bayer grid visible, lens scene fades

### 3. S02 (Bayer) → S03 (Raw Pixels)
**Handoff:** Bayer grid flattens and mosaic dissolves into full-color pixels.

- Progress 0–0.4: Perspective collapses (3D → flat orthographic view)
- Progress 0.4–0.8: Mosaic pattern dissolves — each cell transitions from single-channel Bayer color to full RGB (demosaicing visualization)
- Progress 0.8–1.0: Pixel grid expands from center of Bayer grid

### 4. S03 (Raw) → S04 (RGB)
**Handoff:** Pixel grid splits into three RGB layers like pages of a book.

- Progress 0–0.3: Subtle Z-separation begins between layers
- Progress 0.3–0.7: Three layers peel apart:
  - Red layer slides left + forward
  - Green layer stays center
  - Blue layer slides right + forward
- Progress 0.7–1.0: Layers settle into the three channel canvases of S04

CSS 3D implementation:
```css
/* Driven by JS setting CSS custom properties */
.channel-red { transform: translateX(calc(var(--split) * -120px)) translateZ(calc(var(--split) * 30px)); }
.channel-green { transform: translateZ(calc(var(--split) * 15px)); }
.channel-blue { transform: translateX(calc(var(--split) * 120px)) translateZ(calc(var(--split) * 30px)); }
```

### 5. S04 (RGB) → S05 (YCbCr)
**Handoff:** RGB cards rotate and recolor into YCbCr cards.

- Progress 0–0.5: Cards rotate 180° on Y-axis
- Progress 0.3–0.8: Color transition — canvas content cross-fades from RGB to YCbCr rendering
- Progress 0.8–1.0: Cards settle into YCbCr positions (Y slightly larger/closer)

### 6. S05 (YCbCr) → S06 (Chroma)
**Handoff:** Cb and Cr cards visibly shrink.

- Progress 0–0.4: Cb and Cr cards begin scaling down
- Progress 0.4–0.7: Scale reaches 50% (both X and Y)
- Progress 0.7–1.0: Scaled cards grow back with blurred/interpolated content (showing subsampling effect)

```js
function transitionYcbcrToChroma(p) {
  const shrink = lerp(1, 0.5, easeInOutCubic(Math.min(p * 1.5, 1)));
  cbCard.style.transform = `scale(${shrink})`;
  crCard.style.transform = `scale(${shrink})`;
}
```

### 7. S06 (Chroma) → S07 (Blocks)
**Handoff:** Grid lines sweep across the image.

- Progress 0–0.6: Horizontal lines sweep left-to-right across the image
- Progress 0.3–0.8: Vertical lines sweep top-to-bottom
- Progress 0.8–1.0: Grid locks into place, forming the 8×8 block grid

Implement with canvas overlay — draw grid lines with animated clip paths.

### 8. S07 (Blocks) → S08 (DCT)
**Handoff:** Selected block lifts off the image in 3D.

- Progress 0–0.3: Selected block highlights with glow
- Progress 0.3–0.7: Block lifts on Z-axis, rotates slightly, scales up
- Progress 0.7–1.0: Block expands into the DCT demo panel; spotlight follows it

### 9. S08 (DCT) → S09 (Quant)
**Handoff:** DCT matrix slides left, quantization table slides in from right.

- Progress 0–0.5: DCT coefficient display slides left
- Progress 0.3–0.7: Quantization table slides in from off-screen right
- Progress 0.5–1.0: Division symbol (÷) appears between them; result matrix fades in below

CSS transitions:
```js
function transitionDctToQuant(p) {
  dctPanel.style.transform = `translateX(${lerp(0, -50, p)}%)`;
  quantPanel.style.transform = `translateX(${lerp(100, 0, p)}%)`;
  quantPanel.style.opacity = lerp(0, 1, smoothstep(0.2, 0.6, p));
}
```

### 10. S09 (Quant) → S10 (Zigzag)
**Handoff:** Quantized matrix rotates to face viewer at angle, cursor traces zigzag path.

- Progress 0–0.4: Matrix rotates slightly (CSS rotateX/Y)
- Progress 0.4–0.8: Glowing cursor begins tracing the zigzag path
- Progress 0.8–1.0: 1D sequence starts populating from the bottom-right corner of the matrix

### 11. S10 (Zigzag) → S11 (RLE)
**Handoff:** Runs of zeros compress/collapse.

- Progress 0–0.3: Zoom in on the 1D sequence
- Progress 0.3–0.8: Zero runs visibly collapse — groups of zeros slide together with spring physics
- Progress 0.8–1.0: Compressed RLE tokens settle into place

### 12. S11 (RLE) → S12 (Huffman)
**Handoff:** RLE tokens sprout branches into Huffman tree.

- Progress 0–0.3: Thin lines begin growing upward from each token
- Progress 0.3–0.7: Lines connect and branch, forming the tree structure (root-up construction)
- Progress 0.8–1.0: Full tree visible; token row fades to become the leaf labels

### 13. S12 (Huffman) → S13 (Quality)
**Handoff:** Huffman tree folds into a file icon.

- Progress 0–0.4: Tree nodes converge toward center
- Progress 0.4–0.7: Tree collapses into a single rectangle (file icon shape)
- Progress 0.6–0.8: File icon shrinks dramatically (visualizing compression)
- Progress 0.8–1.0: Icon expands back into the full-quality image of S13

### 14. S13 (Quality) → S14 (Compare)
**Handoff:** Image splits down the middle.

- Progress 0–0.3: A subtle crack appears down the middle of the image
- Progress 0.3–0.7: Left half stays in place, right half shifts slightly right; divider line appears
- Progress 0.7–1.0: Full comparison view with draggable divider

### 15. S14 (Compare) → S15 (Undo)
**Handoff:** Divider rewinds, artifacts dissolve.

- Progress 0–0.5: Divider moves from current position back to right edge
- Progress 0.5–0.8: Compression artifacts visibly dissolve (cross-fade to original)
- Progress 0.8–1.0: Image reconstructs, framing the decode narrative

---

## Easing & Utility Functions

```js
function lerp(a, b, t) { return a + (b - a) * t; }
function easeOutCubic(t) { return 1 - Math.pow(1 - t, 3); }
function easeInOutCubic(t) { return t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2; }
function smoothstep(edge0, edge1, x) {
  const t = Math.max(0, Math.min(1, (x - edge0) / (edge1 - edge0)));
  return t * t * (3 - 2 * t);
}
```

---

## Final Polish

### Performance Optimization
- Ensure all scroll handlers use `requestAnimationFrame` (never raw scroll events)
- Batch DOM reads/writes to avoid layout thrashing
- Use `will-change: transform, opacity` on elements that animate frequently
- Remove `will-change` after animation completes
- Profile and ensure 60fps on Chrome desktop

### Transition Registration System
```js
const transitions = [
  { from: 'hero', to: 's-lens', fn: transitionHeroToLens },
  { from: 's-lens', to: 's-bayer', fn: transitionLensToBayer },
  // ... all 15 transitions
];

function updateTransitions() {
  transitions.forEach(({ from, to, fn }) => {
    const p = getTransitionProgress(from, to);
    if (p > 0 && p < 1) fn(p);
  });
  requestAnimationFrame(updateTransitions);
}
requestAnimationFrame(updateTransitions);
```

### Visual Consistency Check
- All transitions should feel smooth and continuous
- No jarring jumps or pops between sections
- Transition elements should match in style, color, and position across the handoff

### Edge Cases
- Fast scrolling: transitions should still look correct (they track scroll, not time)
- Reverse scrolling: all transitions play backwards smoothly
- Section jumping via sidebar: transitions snap to appropriate state (no animation playback)
- Mobile: simplified transitions where 3D is too expensive (CSS transforms only, no Three.js transitions)

### Accessibility
- `prefers-reduced-motion: reduce`: disable all inter-section transitions; sections simply fade in/out
- All transition elements should be decorative (`aria-hidden="true"`)

---

## Acceptance Criteria

- [ ] All 15 inter-section transitions are implemented
- [ ] Transitions are bound to scroll progress (0→1), not time
- [ ] Scrolling forward plays transitions forward; scrolling back plays them in reverse
- [ ] Transitions feel smooth and continuous at normal scroll speeds
- [ ] Fast scrolling doesn't break transitions
- [ ] Sidebar navigation: transitions snap to correct state without animation
- [ ] Reduced motion: transitions disabled, simple fade-in instead
- [ ] Performance: 60fps maintained during transitions on desktop
- [ ] Mobile: transitions are simplified (CSS only, no Three.js handoffs)
- [ ] No layout thrashing — DOM reads/writes are batched
- [ ] `will-change` is applied during animation and removed after
- [ ] All 17 sections flow into each other as one continuous narrative
- [ ] Page works end-to-end: upload → scroll through all sections → summary
