# Phase 08 — Sections 13–15 & Summary: Output & Decode

**Goal:** Build the final content sections: Live JPEG Compression, Original vs Compressed comparison, Undo JPEG (Decode Pipeline), and the Summary overview.

**Depends on:** Phase 01 (foundation), Phase 02 (uploaded image), Phase 03 (Data Size Bar), Phase 06 (quality slider / quantization)

---

## Section 13 — Live JPEG Compression (`#s-quality`)

### Purpose
Let users see real JPEG compression applied to their full image with an interactive quality slider.

### Text Content
Paragraph 1: Now let's see the complete JPEG pipeline applied to your entire photo. Drag the quality slider to watch the file size change — and notice the visual artifacts that appear at lower quality settings. Below quality 20, you'll start seeing obvious 8×8 block boundaries — that's the block-based processing showing through.

Paragraph 2: Most photos look perfectly acceptable at quality 75–85. Below 50, artifacts become noticeable. The sweet spot depends on the image — photos with lots of smooth gradients (like skies) show banding sooner, while busy textures hide artifacts better.

### Demo — Full-Image Quality Slider

```
┌──────────────────────────────────────┐
│  ┌──────────────────────────────┐    │
│  │                              │    │
│  │  [Compressed image preview]  │    │
│  │                              │    │
│  └──────────────────────────────┘    │
│                                      │
│  Quality: ──────────●───── 75        │
│                                      │
│  File size: 24.3 KB                  │
│  Compression ratio: 62:1             │
│  ▼ Visible artifacts at this level:  │
│    • Slight color banding            │
│    • Minor ringing near edges        │
└──────────────────────────────────────┘
```

**Implementation:**

1. **Compressed image preview:**
   - Use browser's native JPEG encoder: `canvas.toDataURL('image/jpeg', quality/100)`
   - Load the result back into a new `Image`, display on canvas
   - Update on slider change (debounced at 16ms for smooth interaction)

2. **Quality slider:**
   - Range: 1–100, default: 75
   - Shares value with `S.quality` (same as S09)
   - Styled consistently with S09 slider
   - Show numeric value beside the slider

3. **File size display:**
   - Calculate from the `dataURL` length: `Math.round(dataURL.length * 3/4)` bytes (base64 overhead)
   - Display in human-readable format (KB/MB)
   - Rolling counter animation on change

4. **Compression ratio:**
   - `rawSize / compressedSize` — display as "X:1"

5. **Artifact callouts:**
   - Below quality thresholds, show relevant artifact descriptions:
     - Q < 80: "Slight color banding in gradients"
     - Q < 50: "Visible 8×8 block boundaries"
     - Q < 30: "Strong blocking artifacts, color bleeding"
     - Q < 15: "Severe degradation, mosaic-like appearance"

### Data Size Bar
- Stage: "Live quality-adjusted"
- Size: Dynamic — tracks the actual compressed size from the slider
- Color: Dynamic — interpolate based on compression ratio

---

## Section 14 — Original vs Compressed (`#s-compare`)

### Purpose
Side-by-side comparison with a drag divider to directly see compression artifacts.

### Text Content
Paragraph 1: Drag the divider to compare your original photo with the JPEG-compressed version. Pay attention to edges, gradients, and areas of solid color — those are where artifacts are most visible.

Paragraph 2: The four main JPEG artifacts each have a name. Blockiness: visible 8×8 grid. Color banding: smooth gradients become stair-stepped. Ringing: halos around sharp edges. Mosquito noise: shimmering patterns near high-contrast boundaries.

### Demo — Drag-Divider Comparison + Artifact Badges

```
┌──────────────────────────────────────┐
│  ┌──────────────────────────────┐    │
│  │              │               │    │
│  │  Original    │  Compressed   │    │
│  │              │               │    │
│  │           ◄──┼──►            │    │
│  │              │               │    │
│  └──────────────────────────────┘    │
│                                      │
│  Quality: ──────────●───── 75        │
│                                      │
│  ┌─────────┐ ┌───────────┐ ┌──────┐ │
│  │Blockiness│ │Color Band.│ │Ringing│ │  ← Artifact badges
│  └─────────┘ └───────────┘ └──────┘ │
└──────────────────────────────────────┘
```

**Implementation:**

1. **Drag-divider comparison:**
   - Same technique as S06
   - Single canvas: left side = original image, right side = JPEG-compressed image
   - Divider: vertical line with handle, draggable
   - Touch support

2. **Quality slider:**
   - Same shared `S.quality` slider
   - Updates the compressed side in real-time

3. **Artifact badges:**
   - Four clickable/hoverable badges: Blockiness, Color Banding, Ringing, Mosquito Noise
   - Each badge:
     - Pill shape with border
     - On hover/click: shows a tooltip explaining the artifact
     - Visual indicator of severity at current quality (e.g., opacity or color intensity)

### Data Size Bar
- Stage: "Compressed vs. original"
- Size: Dynamic — matches slider
- Dual indicator mode if feasible (show both original and compressed bars)

---

## Section 15 — Undo JPEG: Decode Pipeline (`#s-undo`)

### Purpose
Walk through the JPEG decode process step by step, showing how the compressed data is reconstructed back into an image.

### Text Content
Paragraph 1: When your computer opens a JPEG file, it runs the entire pipeline in reverse. Let's watch each step reconstruct the image — and see exactly where the permanently lost information creates the differences you noticed in the comparison.

Paragraph 2: The lossless steps (Huffman and RLE decoding) recover perfectly — no information was lost there. But dequantization can only multiply by the quantization table values; it can't recover the fractions that were rounded away. That permanent loss is why JPEG is called a "lossy" format.

### Demo — 5-Step Reconstruction Walkthrough

```
┌──────────────────────────────────────┐
│  ┌──────────────────────────────┐    │
│  │                              │    │
│  │  [Reconstruction canvas]     │    │
│  │                              │    │
│  └──────────────────────────────┘    │
│                                      │
│  Step 3 of 5: Dequantize            │
│  ┌──┐ ┌──┐ ┌██┐ ┌──┐ ┌──┐          │
│  │ 1│ │ 2│ │ 3│ │ 4│ │ 5│          │  ← Step indicators
│  └──┘ └──┘ └──┘ └──┘ └──┘          │
│                                      │
│  ◄ Previous    [Apply Step]  Next ►  │
│                                      │
│  "Multiplying quantized values..."   │  ← Step description
└──────────────────────────────────────┘
```

**Implementation:**

### The 5 Decode Steps

| Step | Label | Operation | Description |
|---|---|---|---|
| 1 | Huffman Decode | Reverse Huffman coding | Binary codes → RLE token stream. Perfectly lossless — every bit recovers exactly. |
| 2 | RLE Decode | Expand zero runs | RLE pairs → full 64-value zigzag sequence. All those zeros are restored. |
| 3 | Dequantize | Multiply by quantization table | `value × QT[i]` for each coefficient. Fractions lost during quantization are gone forever — this is where quality loss lives. |
| 4 | Inverse DCT | IDCT transform | Frequency domain → pixel domain. Converts the (approximate) DCT coefficients back into pixel values. |
| 5 | YCbCr → RGB | Color space conversion | Converts back to RGB. Chroma upsampling fills in the subsampled color detail (interpolated, not original). |

### Step Navigation
- 5 step indicator buttons at the bottom (numbered circles)
- "Previous" and "Next" buttons
- "Apply Step" button — triggers the step's operation with animation
- Current step highlighted with accent blue

### Canvas Updates
For each step, show the state of the data on the canvas:
- Step 1: Show the compressed bitstream → decoded RLE tokens (text visualization)
- Step 2: Show expanded zigzag sequence (text visualization)
- Step 3: Show the 8×8 matrix with dequantized values (multiply by QT) — render the block on canvas
- Step 4: Show the IDCT result — render the reconstructed 8×8 block, then reconstruct the full image block-by-block
- Step 5: Show the final RGB image

For the full-image reconstruction in steps 4-5, apply the decode to ALL blocks:

```js
// IDCT (Type-III, 8×8)
function idct2d(coeffs) {
  const N = 8;
  const result = new Float32Array(64);
  for (let x = 0; x < N; x++) {
    for (let y = 0; y < N; y++) {
      let sum = 0;
      for (let u = 0; u < N; u++) {
        const cu = u === 0 ? 1 / Math.sqrt(2) : 1;
        for (let v = 0; v < N; v++) {
          const cv = v === 0 ? 1 / Math.sqrt(2) : 1;
          sum += cu * cv * coeffs[u * N + v] *
            Math.cos((2 * x + 1) * u * Math.PI / (2 * N)) *
            Math.cos((2 * y + 1) * v * Math.PI / (2 * N));
        }
      }
      result[x * N + y] = sum * 0.25 + 128;
    }
  }
  return result;
}
```

### Data Size Bar (Reverse Animation)
The bar **grows** as each decode step is applied:

| Step | Data Size Bar |
|---|---|
| 1 | Grows slightly |
| 2 | Grows |
| 3 | Grows more |
| 4 | Grows significantly |
| 5 | Returns to full raw size (~36 MB) |

Color reverses: red → orange → yellow → green

### Performance Note
Full-image IDCT is expensive (iterating all blocks). Use `setTimeout` chunking:
```js
function processAllBlocks(callback) {
  const blocks = [];
  for (let y = 0; y < S.h; y += 8) {
    for (let x = 0; x < S.w; x += 8) {
      blocks.push({ x, y });
    }
  }
  let i = 0;
  function chunk() {
    const end = Math.min(i + 50, blocks.length);
    for (; i < end; i++) {
      callback(blocks[i].x, blocks[i].y);
    }
    if (i < blocks.length) setTimeout(chunk, 0);
  }
  chunk();
}
```

---

## Summary Section (`#s-summary`)

### Purpose
Recap the entire JPEG pipeline in a visual card grid.

### Layout
```
┌──────────────────────────────────────┐
│  The Complete JPEG Pipeline          │
│                                      │
│  ┌──────┐ ┌──────┐ ┌──────┐         │
│  │ RGB  │ │YCbCr │ │Chroma│         │
│  │      │ │      │ │ Sub  │         │
│  └──────┘ └──────┘ └──────┘         │
│  ┌──────┐ ┌──────┐ ┌──────┐         │
│  │ 8×8  │ │ DCT  │ │Quant │         │
│  └──────┘ └──────┘ └──────┘         │
│  ┌──────┐ ┌──────┐ ┌──────┐         │
│  │Zigzag│ │ RLE  │ │Huffmn│         │
│  └──────┘ └──────┘ └──────┘         │
│  ┌──────────────────────────┐        │
│  │  Final: 36 MB → 300 KB  │        │
│  │  Compression: ~120:1     │        │
│  └──────────────────────────┘        │
└──────────────────────────────────────┘
```

**Implementation:**

1. **10 summary cards** in a CSS grid (auto-fill, min 200px):
   - Each card: `.demo-card` style
   - Icon (from sidebar labels)
   - Step name
   - One-line description
   - "Lossy" or "Lossless" badge
   - Approximate size after this step
   - Clicking a card scrolls to that section

2. **Final stats card** (full width):
   - Raw size → Final JPEG size
   - Total compression ratio
   - "Your photo went from X MB to Y KB — Z× smaller"

3. **Mobile:** Cards in a horizontal scroll carousel

### Cards Data

| Card | Title | Type | One-liner |
|---|---|---|---|
| 1 | RGB Pixels | — | Your photo starts as raw color data |
| 2 | YCbCr | Lossless | Separate brightness from color |
| 3 | Chroma Sub. | Lossy | Shrink color channels by 75% |
| 4 | 8×8 Blocks | — | Divide into independent squares |
| 5 | DCT | Lossless | Convert to frequency patterns |
| 6 | Quantization | Lossy | Round away tiny details |
| 7 | Zigzag | — | Read in diagonal order |
| 8 | RLE | Lossless | Compress runs of zeros |
| 9 | Huffman | Lossless | Shortest codes for common values |
| 10 | JPEG File | — | Final compressed result |

---

## Acceptance Criteria

- [ ] S13: Quality slider compresses image using native JPEG encoder
- [ ] S13: File size and compression ratio update in real-time
- [ ] S13: Artifact descriptions appear at appropriate quality thresholds
- [ ] S14: Drag divider comparison works (mouse + touch)
- [ ] S14: Compressed side updates with quality slider
- [ ] S14: Artifact badges show descriptions on hover/click
- [ ] S15: 5-step decode walkthrough navigates correctly
- [ ] S15: Each step shows appropriate visualization
- [ ] S15: IDCT reconstructs the image (chunked for performance)
- [ ] S15: Data Size Bar reverses (grows, red → green)
- [ ] Summary: 10 cards render in grid layout
- [ ] Summary: Cards are clickable → scroll to section
- [ ] Summary: Final stats show correct compression values
- [ ] Mobile: Summary cards in horizontal carousel
- [ ] All sections update from `S.quality` changes
- [ ] Data Size Bar tracks correctly through S13–S15
