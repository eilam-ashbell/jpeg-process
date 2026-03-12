# Phase 04 — Sections 01–03: Camera to Pixels

**Goal:** Build the first three content sections covering the camera pipeline: Light & Lens, Bayer Sensor, and Raw Pixel Data. 2D demos and explanatory text only — 3D enhancements come in Phase 09.

**Depends on:** Phase 01 (foundation), Phase 02 (uploaded image in `S`), Phase 03 (Data Size Bar updates)

---

## Section 01 — Light Enters the Lens (`#s-lens`)

### Purpose
Explain how light travels through a camera lens to reach the sensor.

### Layout
```
┌──────────────────────────────────────┐
│  01 · Light Enters the Lens          │  ← Section heading
│                                      │
│  [Explanatory text — 2 paragraphs]   │
│                                      │
│  ┌──────────────────────────────┐    │
│  │                              │    │
│  │   [Lens diagram / canvas]    │    │  ← Demo card
│  │                              │    │
│  └──────────────────────────────┘    │
│                                      │
└──────────────────────────────────────┘
```

### Text Content
Paragraph 1: When you press the shutter button, light reflected from the scene enters through the camera lens. The lens is a curved piece of glass that bends (refracts) incoming light rays, focusing them onto a single point — the focal point.

Paragraph 2: Think of the lens as a funnel for light. Millions of photons stream through, each carrying color information from whatever they bounced off. The lens collects all these scattered rays and directs them precisely onto the tiny sensor chip waiting behind it.

### Demo (2D Placeholder)
- A canvas showing a simplified lens diagram:
  - Left side: parallel light rays (lines) coming from the uploaded image (show a small thumbnail)
  - Center: a convex lens shape (two arcs)
  - Right side: rays converge to a focal point, then diverge to a sensor rectangle
- Animated: rays slowly flow left-to-right using `requestAnimationFrame`
- The rays carry colors sampled from the uploaded image

### Data Size Bar
- Stage: "Raw sensor data"
- Size: full raw size (no compression yet)

---

## Section 02 — The Image Sensor & Bayer Mosaic (`#s-bayer`)

### Purpose
Explain how the sensor captures light through a color filter array (Bayer pattern).

### Layout
```
┌──────────────────────────────────────┐
│  02 · The Image Sensor & Bayer       │
│                                      │
│  [Explanatory text — 2 paragraphs]   │
│                                      │
│  ┌──────────────────────────────┐    │
│  │                              │    │
│  │   [Interactive Bayer grid]   │    │  ← Demo card
│  │                              │    │
│  └──────────────────────────────┘    │
│                                      │
│  [Pixel detail callout]             │
│                                      │
└──────────────────────────────────────┘
```

### Text Content
Paragraph 1: Behind the lens sits the image sensor — a grid of millions of tiny light-sensitive cells called photosites. Each photosite can only measure the brightness of light, not its color. So how does a camera capture color?

Paragraph 2: The answer is the Bayer filter — a mosaic of red, green, and blue color filters placed over the sensor. Each photosite sees only one color. Notice there are twice as many green filters as red or blue — that's because human eyes are most sensitive to green, so the camera dedicates extra resolution to it.

### Demo — Interactive Bayer Grid
- Render a portion of the uploaded image as a Bayer mosaic grid on canvas
- Grid size: show a ~16×16 region of photosites (zoomed in)
- Each cell is colored with its Bayer filter color (R, G, or B) at reduced opacity, overlaid on the actual pixel value
- RGGB pattern: Row 0 = R,G,R,G... Row 1 = G,B,G,B... (repeating)

**Interaction:**
- Click on any cell: highlight it, show a callout with:
  - Position (row, col)
  - Filter color (Red / Green / Blue)
  - Raw intensity value (0–255)
  - "This photosite only sees [color] light"
- Hover: subtle highlight on the cell

### Text Topics
- Photosites
- Bayer mosaic pattern (RGGB)
- Why green is doubled
- Each photosite is "colorblind" — sees only one color

---

## Section 03 — Raw Pixel Data (`#s-raw`)

### Purpose
Show what raw pixel data looks like after demosaicing, and explain the data size.

### Layout
```
┌──────────────────────────────────────┐
│  03 · Raw Pixel Data                 │
│                                      │
│  [Explanatory text — 2 paragraphs]   │
│                                      │
│  ┌─────────────┬─────────────────┐   │
│  │ Source image │ Zoomed pixel    │   │  ← Dual-panel demo card
│  │ (click to   │ grid showing    │   │
│  │  zoom)      │ individual RGB  │   │
│  │             │ values          │   │
│  └─────────────┴─────────────────┘   │
│                                      │
│  Each pixel = 3 bytes (R, G, B)      │
│  Your photo: WxH = N pixels = X MB   │  ← Stats callout
│                                      │
└──────────────────────────────────────┘
```

### Text Content
Paragraph 1: After demosaicing — a clever algorithm that fills in the missing color values for each photosite — every pixel now has a full set of three numbers: Red, Green, and Blue. These three values, each ranging from 0 to 255, mix together to create the final color you see.

Paragraph 2: At this point, your photo is just a massive grid of numbers. Each pixel takes 3 bytes of storage (one per channel). For a typical 12-megapixel photo, that's over 36 million bytes — about 36 MB of raw data. JPEG's job is to shrink this dramatically without ruining the image.

### Demo — Dual-Panel Pixel Inspector
**Left panel — Source image:**
- Render the uploaded image on a canvas
- Crosshair cursor
- On click: capture the click coordinates and update the right panel

**Right panel — Zoomed pixel grid:**
- Show a ~8×8 or 10×10 grid of pixels centered on the clicked location
- Each pixel cell is large enough to see individually (~30px)
- Each cell shows the pixel color as background
- On hover over a cell: show RGB values as a tooltip or label below the grid
  - Format: `R: 182  G: 97  B: 45`

**Stats callout:**
- Display computed values:
  - `"Your photo: {S.w} × {S.h} = {S.w * S.h} pixels"`
  - `"Uncompressed: {(S.w * S.h * 3 / 1_000_000).toFixed(1)} MB"`
  - Contextualize: `"That's {Math.round(S.w * S.h * 3 / 300_000)}× larger than the final JPEG"`

### Data Size Bar
- Stage: "Full RGB pixels"
- Size: `S.w * S.h * 3` bytes

---

## Shared Implementation Notes

### Section Heading Pattern
Each section follows the same heading pattern:
```html
<div class="section-header reveal">
  <span class="section-number">01</span>
  <h2 class="section-title">Light Enters the Lens</h2>
</div>
```

Style the section number in `var(--accent-blue)`, smaller font, above the title.

### Demo Card Pattern
All demo cards use the `.demo-card` class from Phase 01.

### Canvas Rendering
- All canvases should use `willReadFrequently: true` context option where `getImageData` is needed
- Scale canvas CSS size to `width: 100%` but keep internal resolution fixed
- Handle window resize: redraw on resize

### Initialization
- All demos initialize on `imageReady` event
- Each section registers itself: `document.addEventListener('imageReady', initSection02)`
- Guard against missing image: check `S.img` before accessing

---

## Acceptance Criteria

- [ ] S01: Lens diagram renders and animates light rays
- [ ] S02: Bayer grid renders with correct RGGB pattern from uploaded image
- [ ] S02: Clicking a cell shows photosite details
- [ ] S03: Clicking source image updates the zoomed pixel grid
- [ ] S03: Pixel inspector shows correct RGB values
- [ ] S03: Stats callout shows correct computed values
- [ ] All sections have properly styled headings and body text
- [ ] All demos are in `.demo-card` containers
- [ ] All sections respond to `imageReady` event
- [ ] Data Size Bar updates correctly for each section
- [ ] Responsive layout works on mobile (single column, canvases scale)
- [ ] All sections have `.reveal` animation on scroll
