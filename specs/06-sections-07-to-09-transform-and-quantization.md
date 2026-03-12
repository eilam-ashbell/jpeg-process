# Phase 06 — Sections 07–09: Transform & Quantization

**Goal:** Build the core compression sections: 8×8 Block Division, Discrete Cosine Transform (DCT), and Quantization. These sections are the heart of JPEG compression.

**Depends on:** Phase 01 (foundation), Phase 02 (image), Phase 03 (Data Size Bar), Phase 05 (YCbCr data)

---

## Section 07 — 8×8 Block Division (`#s-blocks`)

### Purpose
Show how JPEG divides the image into 8×8 pixel blocks for independent processing.

### Text Content
Paragraph 1: JPEG doesn't compress the whole image at once. Instead, it divides your photo into a grid of tiny 8×8 pixel blocks — just 64 pixels each. For a 1080p image, that's over 32,000 blocks, each processed independently.

Paragraph 2: Why 8×8? It's a sweet spot — small enough to capture local patterns efficiently, but large enough to contain meaningful visual information. Every block goes through the same compression pipeline from here on. Click any block to follow its journey through the next steps.

### Demo — Clickable Block Selector

```
┌──────────────────────────────────────┐
│  ┌────────────────────────────────┐  │
│  │                                │  │
│  │  [Image with 8×8 grid overlay] │  │  ← Clickable
│  │                                │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  Selected Block Detail         │  │
│  │  Block (3, 5) — Position       │  │
│  │  ┌─────────┐                   │  │
│  │  │ 8×8     │  Y:  [values]     │  │
│  │  │ zoomed  │  Cb: [values]     │  │
│  │  │ block   │  Cr: [values]     │  │
│  │  └─────────┘                   │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Implementation:**

1. **Image with grid overlay:**
   - Render uploaded image on canvas
   - Draw 8×8 grid lines over it (1px, `rgba(37, 99, 235, 0.3)`)
   - On hover: highlight the hovered block with `rgba(37, 99, 235, 0.15)` fill
   - On click: select the block — highlight with accent blue border (2px), update `S.selBx`, `S.selBy`
   - Selected block stays highlighted with a pulsing glow

2. **Block detail panel:**
   - Show zoomed view of the selected 8×8 block (each pixel ~30px)
   - Display the YCbCr values for the block:
     - Extract 64 Y values, 64 Cb values, 64 Cr values from the selected block position
     - Store in `S.block = { y: [...], cb: [...], cr: [...] }`
   - Show as a small matrix grid (8×8) with values visible

3. **Spacebar interaction:**
   - Pressing Spacebar selects a random block
   - Add event listener: `document.addEventListener('keydown', e => { if (e.code === 'Space') selectRandomBlock(); })`

**Block extraction:**
```js
function extractBlock(channel, x, y, w) {
  const block = new Float32Array(64);
  for (let row = 0; row < 8; row++) {
    for (let col = 0; col < 8; col++) {
      block[row * 8 + col] = channel[(y + row) * w + (x + col)];
    }
  }
  return block;
}
```

### Data Size Bar
- Stage: "Divided into blocks"
- Size: same as after subsampling (~18 MB)

---

## Section 08 — Discrete Cosine Transform (DCT) (`#s-dct`)

### Purpose
Explain and visualize how DCT converts pixel values into frequency coefficients.

### Text Content
Paragraph 1: Here's where the math magic happens. The Discrete Cosine Transform takes each 8×8 block of pixel values and converts them into 64 frequency values — think of it like a graphic equalizer for images. Low frequencies (top-left) capture smooth gradients, high frequencies (bottom-right) capture sharp edges and fine details.

Paragraph 2: The brilliant insight: most of the visual energy in a typical photo block concentrates in just a few low-frequency values. The high-frequency coefficients are often tiny or zero. This is why JPEG can throw away data without you noticing — it's discarding frequencies your eyes barely register.

### Demo — Pixel Matrix + DCT Coefficient Matrix

```
┌──────────────────────────────────────┐
│  ┌──────────┐      ┌──────────┐     │
│  │ Pixel    │  DCT │ Frequency│     │
│  │ values   │  ──► │ coeffs   │     │
│  │ (8×8)    │      │ (8×8)    │     │
│  └──────────┘      └──────────┘     │
│                                      │
│  [ ▶ Animate Transform ]            │  ← Button
│                                      │
│  DC coefficient: 1024               │
│  Non-zero AC coefficients: 12/63     │
│  Energy in top-left quadrant: 94%    │  ← Stats
└──────────────────────────────────────┘
```

**Implementation:**

1. **Pixel matrix (left):**
   - 8×8 grid showing the Y-channel values from the selected block
   - Each cell: background color = grayscale shade, text = numeric value (0–255)
   - Font: JetBrains Mono, small size

2. **DCT coefficient matrix (right):**
   - 8×8 grid showing DCT coefficients
   - Cell color: magnitude-based heatmap (|value| mapped to color intensity)
   - High values = bright, zero/low values = dark/muted
   - Top-left cell (DC) typically largest — highlight it
   - Font: JetBrains Mono

3. **Animate Transform button:**
   - On click: animate the transition from pixel domain to frequency domain
   - Animation: cells morph their values (rolling numbers) and colors over ~1.5s
   - Button text changes to "Reset" after animation, allows replaying

4. **Stats below:**
   - DC coefficient value
   - Count of non-zero AC coefficients (out of 63)
   - Energy concentration metric

**DCT Algorithm (Type-II, 8×8):**
```js
function dct2d(block) {
  const N = 8;
  const result = new Float32Array(64);
  for (let u = 0; u < N; u++) {
    for (let v = 0; v < N; v++) {
      let sum = 0;
      const cu = u === 0 ? 1 / Math.sqrt(2) : 1;
      const cv = v === 0 ? 1 / Math.sqrt(2) : 1;
      for (let x = 0; x < N; x++) {
        for (let y = 0; y < N; y++) {
          sum += (block[x * N + y] - 128) *
            Math.cos((2 * x + 1) * u * Math.PI / (2 * N)) *
            Math.cos((2 * y + 1) * v * Math.PI / (2 * N));
        }
      }
      result[u * N + v] = 0.25 * cu * cv * sum;
    }
  }
  return result;
}
```

Store result: `S.dctC = dct2d(S.block.y);`

### Data Size Bar
- Stage: "After DCT"
- Size: same (~18 MB) — DCT is a transform, not compression

---

## Section 09 — Quantization (`#s-quant`)

### Purpose
Show how quantization reduces precision, creating zeros and enabling real compression.

### Text Content
Paragraph 1: Quantization is where JPEG actually loses information — permanently. Each DCT coefficient is divided by a number from the quantization table, then rounded to the nearest integer. Large divisors for high-frequency coefficients crush small values to zero, while the important low-frequency values survive.

Paragraph 2: The quality slider you see on image editors controls exactly this step. Quality 100 means gentle quantization (large file, high quality). Quality 1 means aggressive quantization (tiny file, blocky mess). At quality 75 — a common default — about 70% of the coefficients become zero. That's a lot of data you no longer need to store.

### Demo — Dual-Panel Matrix + Quality Slider

```
┌──────────────────────────────────────┐
│  ┌──────────┐ ÷  ┌──────────┐       │
│  │ DCT      │    │ Quant    │       │
│  │ coeffs   │    │ table    │       │
│  │ (8×8)    │    │ (8×8)    │       │
│  └──────────┘    └──────────┘       │
│              ═                       │
│        ┌──────────┐                  │
│        │ Result   │                  │
│        │ (8×8)    │  ← many zeros   │
│        └──────────┘                  │
│                                      │
│  Quality: ────────●──── 75           │
│                                      │
│  Zeros: 45/64 (70%)                 │
│  Effective compression: ~8:1         │
└──────────────────────────────────────┘
```

**Implementation:**

1. **Three matrices:**
   - **DCT coefficients** (from S.dctC)
   - **Quantization table** (standard JPEG luminance table, scaled by quality)
   - **Result** = `round(DCT[i] / QT[i])` for each cell

2. **Quality slider:**
   - Range: 1–100, default 75
   - Updates the quantization table and result in real-time
   - Shared with `S.quality` — other sections (S13) use this too
   - Slider styled: track = `var(--border)`, thumb = `var(--accent-blue)`, filled portion = `var(--accent-blue-tint)`

3. **Visual emphasis on zeros:**
   - Zero cells in the result matrix: dimmed (low opacity or very light background)
   - Non-zero cells: bold text, slightly lifted appearance

4. **Stats:**
   - Count of zeros out of 64
   - Percentage
   - Approximate compression ratio

**Standard JPEG Luminance Quantization Table:**
```js
const BASE_QT = [
  16,11,10,16,24,40,51,61,
  12,12,14,19,26,58,60,55,
  14,13,16,24,40,57,69,56,
  14,17,22,29,51,87,80,62,
  18,22,37,56,68,109,103,77,
  24,35,55,64,81,104,113,92,
  49,64,78,87,103,121,120,101,
  72,92,95,98,112,100,103,99
];
```

**Quality scaling formula:**
```js
function scaleQT(quality) {
  const scale = quality < 50 ? Math.floor(5000 / quality) : 200 - quality * 2;
  return BASE_QT.map(v => Math.max(1, Math.min(255, Math.round((v * scale + 50) / 100))));
}
```

**Quantize:**
```js
function quantize(dctCoeffs, qt) {
  return dctCoeffs.map((v, i) => Math.round(v / qt[i]));
}
```

Store: `S.quantC = quantize(S.dctC, scaleQT(S.quality));`

### Data Size Bar
- Stage: "After quantization"
- Size: ~2–4 MB (dramatically smaller — most data is now zeros)
- Color: Orange

---

## Shared Notes

### Block Update Chain
When the selected block changes (S07 click or Spacebar):
1. Extract block from YCbCr channels → `S.block`
2. Run DCT on Y channel → `S.dctC`
3. Run quantization → `S.quantC`
4. Subsequent sections (S10, S11, S12) also update — emit a custom event `'blockChanged'`

```js
function updateBlock(bx, by) {
  S.selBx = bx;
  S.selBy = by;
  S.block = {
    y: extractBlock(S.yChannel, bx, by, S.w),
    cb: extractBlock(S.cbChannel, bx, by, S.w),
    cr: extractBlock(S.crChannel, bx, by, S.w),
  };
  S.dctC = dct2d(S.block.y);
  S.quantC = quantize(S.dctC, scaleQT(S.quality));
  document.dispatchEvent(new CustomEvent('blockChanged'));
}
```

### Quality Change Chain
When quality slider changes:
1. Recompute quantization table
2. Re-quantize → `S.quantC`
3. Update S10, S11, S12 as well → dispatch `'qualityChanged'` event

### Matrix Cell Styling
All 8×8 matrix displays share common CSS:
```css
.matrix-grid {
  display: grid;
  grid-template-columns: repeat(8, 1fr);
  gap: 1px;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.72em;
}
.matrix-cell {
  aspect-ratio: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2px;
}
```

---

## Acceptance Criteria

- [ ] S07: 8×8 grid overlay renders correctly on the image
- [ ] S07: Clicking a block selects it and updates `S.selBx`, `S.selBy`
- [ ] S07: Block detail shows zoomed view and YCbCr values
- [ ] S07: Spacebar selects a random block
- [ ] S08: Pixel matrix displays correct Y values for selected block
- [ ] S08: DCT algorithm produces correct coefficients
- [ ] S08: DCT coefficient matrix renders with heatmap coloring
- [ ] S08: Animate button morphs values from pixels to frequencies
- [ ] S09: Quantization table renders correctly
- [ ] S09: Quality slider updates quantization in real-time
- [ ] S09: Zero cells are visually distinguished
- [ ] S09: Stats (zero count, compression ratio) update correctly
- [ ] Block change propagates through S07→S08→S09 chain
- [ ] Quality change propagates from S09 to dependent sections
- [ ] Data Size Bar updates correctly for each section
- [ ] All sections responsive on mobile
