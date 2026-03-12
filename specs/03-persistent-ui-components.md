# Phase 03 — Persistent UI Components (Data Size Bar & Sidebar Navigation)

**Goal:** Build the two persistent UI elements that remain visible throughout the scrolling experience: the Data Size Bar (bottom) and the collapsible Sidebar Navigation (right).

**Depends on:** Phase 01 (scroll infrastructure, design system), Phase 02 (upload / image dimensions)

---

## Deliverable

1. **Data Size Bar** — fixed bottom bar showing current virtual file size
2. **Sidebar Navigation** — collapsible right-side panel for section jumping

Both components appear only after the user has uploaded an image.

---

## Part A: Data Size Bar

### Position & Style
- Fixed, bottom of viewport, full width
- `z-index: 1000`
- Height: `52px` collapsed, `72px` on hover
- Background: `rgba(255,255,255,0.85)` with `backdrop-filter: blur(12px)`
- Top border: `1px solid var(--border)`
- Hidden until image is uploaded (fade in on `imageReady` event)

### Contents (left to right)

```
┌─────────────────────────────────────────────────────────┐
│  Raw pixels  ████████████████████████████░░░  36.0 MB   │
└─────────────────────────────────────────────────────────┘
```

1. **Stage label** — text showing current pipeline stage (e.g., "Raw pixels", "After YCbCr", "After Quantization")
2. **Animated fill bar** — a horizontal bar that shrinks as compression progresses
   - Color transitions: green → yellow → orange → red through the spectrum
   - Width represents relative size (100% = raw, proportional to compressed)
3. **Numeric label** — file size with rolling counter animation (e.g., "36.0 MB" → "245 KB")
4. **Compression ratio badge** — appears when significant compression occurs (e.g., "147× smaller")

### Size Values Per Section

Scale these values proportionally based on actual uploaded image pixel count. Base values are for a 12MP photo (~36 MB raw).

| Section | Stage Label | Approx. Size | Bar % | Color |
|---|---|---|---|---|
| Hero | Raw sensor data | ~36 MB | 100% | Green |
| S01 | After demosaicing | ~36 MB | 100% | Green |
| S02–S03 | Full RGB pixels | ~36 MB | 100% | Green |
| S04 | RGB channels | ~36 MB | 100% | Green |
| S05 | After YCbCr conversion | ~36 MB | 100% | Green |
| S06 | After 4:2:0 subsampling | ~18 MB | 50% | Yellow |
| S07 | Divided into blocks | ~18 MB | 50% | Yellow |
| S08 | After DCT | ~18 MB | 50% | Yellow |
| S09 | After quantization | ~2–4 MB | 8% | Orange |
| S10 | After zigzag scan | ~2–4 MB | 8% | Orange |
| S11 | After RLE | ~800 KB | 2.2% | Red |
| S12 | After Huffman coding | ~300 KB | 0.8% | Red |
| S13 | Live (quality-adjusted) | Dynamic | Dynamic | Dynamic |
| S14 | Compressed vs. original | Dynamic | Dynamic | Dual |
| S15 | Decoding (rebuilding) | Grows → 36 MB | Grows | Green |
| Summary | Final JPEG on disk | ~300 KB | 0.8% | Red |

### Scaling Formula
```js
const basePixels = 12_000_000; // 12MP reference
const actualPixels = S.w * S.h;
const scaleFactor = actualPixels / basePixels;
// Multiply all base sizes by scaleFactor
```

### Animation Behavior
- Size value: **rolling counter** effect — digits scroll up/down individually, duration 600ms
- Fill bar width: `transition: width 800ms cubic-bezier(0.4, 0, 0.2, 1)`
- Color: smooth transition through green→yellow→orange→red
- On S15 (Undo): bar **reverses** — animates from red back to green
- Subtle pulse (`scale 1.0 → 1.02 → 1.0`) on significant size changes

### Interaction
- Clicking the bar expands a tooltip overlay showing size at each step
- Tooltip: list of all stages with their sizes, current stage highlighted
- Click outside or click bar again to dismiss

### Mobile
- Height: `40px`
- Abbreviated labels (e.g., "Raw" instead of "Raw sensor data")
- Smaller font size

### Accessibility
- `aria-live="polite"` on the numeric label so screen readers announce changes
- `role="status"` on the bar container

### Update Mechanism
- Listen to scroll progress changes
- Determine which section is currently active (most visible / scroll position)
- Update stage label, size, bar width, and color accordingly
- Expose `updateDataBar(sectionId)` function for other phases to call

---

## Part B: Sidebar Navigation

### Toggle Tab (Default/Collapsed State)
- Fixed, right edge of viewport, vertically centered
- `z-index: 900`
- Size: `32×80px`
- Label: `"≡ Sections"` rotated 90° (`writing-mode: vertical-rl` or CSS rotate)
- Background: `var(--accent-blue)`, text white
- Border radius: `14px 0 0 14px` (rounded on left side)
- Cursor: pointer
- Hidden until image is uploaded

### Expanded Panel
- Width: `220px`, slides in from right edge
- Background: `rgba(255,255,255,0.92)` with `backdrop-filter: blur(16px)`
- Left border: `1px solid var(--border)`
- Border radius: `14px 0 0 14px`
- Transition: `transform 0.3s ease` (translateX)

### Panel Header
- `"§ Sections"` + close button (`×`)
- Bottom border separator

### Navigation Items

Each item is a row with:
- Status indicator: `○` (not reached), `●` (current), `✓` (visited/passed)
- Section number (or icon for Hero/Summary)
- Short label

```
| Entry   | Short Label        | Icon |
|---------|--------------------|------|
| Hero    | Upload Your Photo  | 📷   |
| S01     | Light & Lens       | 🔆   |
| S02     | Bayer Sensor       | 🟩   |
| S03     | Raw Pixels         | 🔍   |
| S04     | RGB Channels       | 🎨   |
| S05     | YCbCr Space        | 🌈   |
| S06     | Chroma Subsampling | ⬇️   |
| S07     | 8×8 Blocks         | 🧩   |
| S08     | DCT Transform      | 〰️   |
| S09     | Quantization       | ✂️   |
| S10     | Zigzag Scan        | ↗️   |
| S11     | Run-Length Enc.    | 🗜️   |
| S12     | Huffman Coding     | 🌳   |
| S13     | Live Quality       | 🎛️   |
| S14     | Comparison         | ⟺   |
| S15     | Undo JPEG          | 🔄   |
| Summary | Full Pipeline      | ★    |
```

### Behavior
- **Toggle:** Click tab or press `N` key
- **Close:** Click outside, press `Escape`, or press `N` again
- **Active section:** Highlighted with accent blue background + white text, updated via IntersectionObserver
- **Visited sections:** Small checkmark icon; unvisited show empty dot
- **Click item:** Smooth-scroll to that section via `element.scrollIntoView({ behavior: 'smooth' })`, then close panel
- **Progress indicator:** Thin vertical line on left edge of panel — filled portion grows as user scrolls

### Keyboard Navigation
- Arrow keys move focus between items when panel is open
- Enter jumps to selected section
- Focus trap when panel is open (Tab cycles within panel)

### Mobile
- Panel opens to `100vw` (full screen width)
- Larger tap targets: `48px` row height
- Close button more prominent

---

## Integration Notes

- Both components listen for `imageReady` event to show themselves
- Both update based on scroll position (use `S.sectionProgress`)
- Data Size Bar accounts for the `52px` bar height in bottom padding of the page (add `padding-bottom: 60px` to the last section or body)
- Sidebar toggle should not overlap with Data Size Bar (position the tab above the bar's height)

---

## Acceptance Criteria

- [ ] Data Size Bar appears after image upload, fixed at bottom
- [ ] Stage label updates as user scrolls through sections
- [ ] Fill bar width and color animate smoothly on section change
- [ ] Rolling counter animation works on size changes
- [ ] Clicking bar shows breakdown tooltip
- [ ] Bar is accessible (aria-live, role=status)
- [ ] Sidebar toggle tab visible on right edge after upload
- [ ] Clicking tab opens the navigation panel
- [ ] Panel shows all 17 sections with correct labels
- [ ] Active section is highlighted and updates on scroll
- [ ] Clicking a nav item scrolls to that section and closes panel
- [ ] `N` key toggles panel; `Escape` closes it
- [ ] Arrow keys navigate within open panel
- [ ] Focus is trapped in open panel
- [ ] Mobile: bar is 40px, sidebar is full-width
- [ ] Both components respect reduced-motion preference
