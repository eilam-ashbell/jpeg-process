# Phase 05 — Sections 04–06: Color Processing

**Goal:** Build the color processing pipeline sections: RGB Channel decomposition, YCbCr color space conversion, and Chroma Subsampling. All 2D canvas demos — 3D enhancements in Phase 09.

**Depends on:** Phase 01 (foundation), Phase 02 (uploaded image), Phase 03 (Data Size Bar), Phase 04 (section patterns)

---

## Section 04 — RGB Color Channels (`#s-rgb`)

### Purpose
Show how an image is composed of separate Red, Green, and Blue channels.

### Text Content
Paragraph 1: Every pixel in your photo is a blend of three colors: red, green, and blue. By separating these channels, you can see each color's individual contribution. Look at the green channel — it's the brightest because it carries most of the detail your eyes care about.

Paragraph 2: JPEG processes these channels, but not in RGB form. The first clever trick is converting to a different color space that's better suited for compression. But first, let's see what RGB separation looks like.

### Demo — 4-Panel Channel Display

```
┌────────────┬────────────┐
│  Combined  │  Red       │
│  (full     │  channel   │
│  color)    │  (grayscale │
│            │  of R vals) │
├────────────┼────────────┤
│  Green     │  Blue      │
│  channel   │  channel   │
└────────────┴────────────┘
```

**Implementation:**
- Four canvases in a 2×2 CSS grid
- **Combined:** The original uploaded image
- **Red channel:** For each pixel, set `R = original R, G = 0, B = 0` (shows red intensity as red tones). Alternatively, show as grayscale: `R = G = B = originalR`
- **Green channel:** Same approach with green values
- **Blue channel:** Same approach with blue values
- Use grayscale representation (more educational — shows intensity clearly)
- Label each canvas with its channel name

**Mobile layout:**
- Horizontal scroll carousel with `scroll-snap-type: x mandatory`
- Each panel is full-width within the carousel
- Dot indicators below showing current panel (4 dots)

### Data Size Bar
- Stage: "RGB channels"
- Size: same as raw (~36 MB for 12MP) — no compression yet

---

## Section 05 — YCbCr Color Space (`#s-ycbcr`)

### Purpose
Explain the conversion from RGB to YCbCr and why it matters for compression.

### Text Content
Paragraph 1: JPEG doesn't work with red, green, and blue directly. Instead, it converts your image into a different color system called YCbCr. Y captures the brightness (luminance) — essentially a black-and-white version of your photo. Cb and Cr capture the color information (chrominance) — the blue-difference and red-difference signals.

Paragraph 2: Why bother? Because your eyes are much more sensitive to brightness than to color. By separating these, JPEG can aggressively compress the color information (Cb, Cr) while keeping the brightness (Y) sharp — and you'll barely notice the difference.

### Conversion Formulas
Display in a code block (JetBrains Mono):
```
Y  =  0.299 × R  +  0.587 × G  +  0.114 × B
Cb = −0.169 × R  −  0.331 × G  +  0.500 × B  +  128
Cr =  0.500 × R  −  0.419 × G  −  0.081 × B  +  128
```

Style: inside a `.demo-card` with subtle background, monospace font.

### Demo — 3-Panel YCbCr Display

```
┌──────────────┬──────────────┬──────────────┐
│  Y (Luma)    │  Cb (Blue-   │  Cr (Red-    │
│  Grayscale   │  diff)       │  diff)       │
│  brightness  │  Cool tones  │  Warm tones  │
└──────────────┴──────────────┴──────────────┘
```

**Implementation:**
- Three canvases in a row (CSS grid or flexbox)
- **Y channel:** Compute `Y = 0.299*R + 0.587*G + 0.114*B` for each pixel, display as grayscale
- **Cb channel:** Compute `Cb = -0.168736*R - 0.331264*G + 0.5*B + 128`, display as grayscale (0–255 mapped). Optionally use a blue-tinted colormap for visual clarity.
- **Cr channel:** Compute `Cr = 0.5*R - 0.418688*G - 0.081312*B + 128`, display as grayscale. Optionally red-tinted colormap.
- Label each with channel name and brief description

**Store YCbCr data** for later use:
```js
// Store full-image YCbCr arrays for use in S06+
S.yChannel = new Uint8Array(S.w * S.h);
S.cbChannel = new Uint8Array(S.w * S.h);
S.crChannel = new Uint8Array(S.w * S.h);
```

**Mobile layout:**
- Horizontal scroll carousel with snap, same as S04
- 3 panels, dot indicators

### Data Size Bar
- Stage: "After YCbCr conversion"
- Size: same as raw — conversion doesn't change data size, it's just a transformation

---

## Section 06 — Chroma Subsampling 4:2:0 (`#s-chroma`)

### Purpose
Show how JPEG reduces color resolution by subsampling chroma channels, achieving the first real size reduction.

### Text Content
Paragraph 1: Here's where compression actually begins. Since your eyes can't detect fine color detail, JPEG shrinks the Cb and Cr channels to half their resolution in both width and height. This is called 4:2:0 chroma subsampling — and it instantly cuts the color data by 75%.

Paragraph 2: The result? Your image goes from needing about 36 MB to about 18 MB — a 50% reduction — and the difference is nearly invisible. The brightness (Y) channel stays at full resolution, preserving all the sharp detail your eyes are most sensitive to.

### Demo — Drag-Divider Comparison

```
┌──────────────────────────────────────┐
│                                      │
│  Original  │◄ drag ►│  Subsampled    │
│            │        │                │
│            │        │                │
└──────────────────────────────────────┘
        Quality: ──●────── 4:2:0
```

**Implementation:**

1. **Left side (Original):** Full-resolution image reconstructed from YCbCr (or just the original)
2. **Right side (Subsampled):**
   - Subsample Cb and Cr: for each 2×2 block of pixels, average the 4 Cb values into 1, same for Cr
   - Reconstruct: upsample Cb/Cr back to full size (nearest-neighbor or bilinear), convert back to RGB
   - This shows the quality loss from chroma subsampling

3. **Drag divider:**
   - A vertical line that the user can drag left/right
   - Use a single canvas; draw original on left of divider, subsampled on right
   - Divider: 2px line, accent blue, with a circular handle (16px diameter) in the center
   - CSS cursor: `col-resize` on the handle
   - Touch support: handle `touchstart`, `touchmove`, `touchend`

**4:2:0 Subsampling Algorithm:**
```js
function subsample420(cb, cr, w, h) {
  const halfW = w / 2, halfH = h / 2;
  const cbSub = new Uint8Array(halfW * halfH);
  const crSub = new Uint8Array(halfW * halfH);
  for (let y = 0; y < h; y += 2) {
    for (let x = 0; x < w; x += 2) {
      const i = y * w + x;
      cbSub[(y/2) * halfW + (x/2)] = (cb[i] + cb[i+1] + cb[i+w] + cb[i+w+1]) / 4;
      crSub[(y/2) * halfW + (x/2)] = (cr[i] + cr[i+1] + cr[i+w] + cr[i+w+1]) / 4;
    }
  }
  return { cbSub, crSub, halfW, halfH };
}
```

### Data Size Bar
- Stage: "After 4:2:0 subsampling"
- Size: ~50% of raw (Y at full res + Cb/Cr at quarter res = 1 + 0.25 + 0.25 = 1.5 / 3 = 50%)
- Color: Yellow

---

## Shared Notes

### Mobile Carousels
For S04 and S05, implement horizontal scroll carousels on screens < 680px:
```css
@media (max-width: 680px) {
  .channel-grid {
    display: flex;
    overflow-x: auto;
    scroll-snap-type: x mandatory;
    -webkit-overflow-scrolling: touch;
  }
  .channel-grid > .channel-panel {
    flex: 0 0 85vw;
    scroll-snap-align: start;
  }
}
```

Add dot indicators below each carousel:
- One dot per panel
- Active dot: `var(--accent-blue)`, others: `var(--border)`
- Update active dot on scroll using `IntersectionObserver` within the carousel container

### Channel Canvas Rendering
All channel canvases should:
- Share the same dimensions as the uploaded image
- Use `CSS width: 100%` to scale responsively
- Re-render on `imageReady` event

---

## Acceptance Criteria

- [ ] S04: Four channel canvases render correctly (combined, R, G, B)
- [ ] S04: Channels show correct isolated color data
- [ ] S05: YCbCr conversion produces correct Y, Cb, Cr channels
- [ ] S05: Conversion formulas are displayed in monospace
- [ ] S05: YCbCr data is stored in global state for S06
- [ ] S06: 4:2:0 subsampling algorithm works correctly
- [ ] S06: Drag divider comparison shows original vs subsampled
- [ ] S06: Divider works with mouse and touch
- [ ] Mobile: S04 and S05 use horizontal carousels with snap
- [ ] Mobile: Dot indicators show and update correctly
- [ ] Data Size Bar updates for each section (same size for S04/S05, 50% for S06)
- [ ] All text content is present and properly styled
- [ ] All demos initialize on `imageReady` event
