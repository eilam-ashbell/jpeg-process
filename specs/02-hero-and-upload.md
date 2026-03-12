# Phase 02 — Hero Section & Image Upload

**Goal:** Build the hero/landing section with image upload functionality, image processing pipeline, and the initial visual presentation. No 3D yet — that comes in Phase 09.

**Depends on:** Phase 01 (foundation, design system, global state)

---

## Deliverable

Populate the `#hero` section with:
1. Upload UI (click + drag-drop)
2. Image processing (scale, pad, store in global state)
3. Hero visual layout (title, subtitle, upload zone)
4. Post-upload state change (upload zone transforms to show the image)

---

## 1. Hero Layout

```
┌──────────────────────────────────────────┐
│                                          │
│         Inside JPEG                      │  ← Page title (Inter 900)
│  How your photos get compressed          │  ← Subtitle (text-secondary)
│                                          │
│  ┌──────────────────────────────────┐    │
│  │                                  │    │
│  │   📷 Drop your photo here       │    │  ← Upload zone
│  │   or click to browse             │    │
│  │                                  │    │
│  └──────────────────────────────────┘    │
│                                          │
│  Your photo never leaves your device.    │  ← Privacy note (text-muted)
│                                          │
└──────────────────────────────────────────┘
```

- Title: `Inside JPEG` — use the page title typography
- Subtitle: `Scroll through the journey of a JPEG, step by step — using your own photo.`
- Upload zone: dashed border (`2px dashed var(--border)`), rounded corners (`var(--radius)`), centered, `max-width: 400px`, `min-height: 200px`
- Upload zone hover: border color changes to `var(--accent-blue)`, subtle background tint
- Upload zone drag-over: same as hover state + scale(1.02) transform

---

## 2. Upload Handling

### Input
- Hidden `<input type="file" accept="image/*">`
- Click on upload zone triggers file input
- Drag-and-drop on upload zone
- Accept any `image/*` type

### Image Processing Pipeline

On file selection:

1. **Read file** via `FileReader` as `dataURL`
2. **Create Image object**, wait for `onload`
3. **Scale to fit 480×480px** — maintain aspect ratio, fit within 480×480 bounding box
4. **Pad to nearest 8×8 multiple** — add black pixels if width or height isn't divisible by 8
5. **Draw to canvas**, extract `ImageData`
6. **Store in global state:**
   ```js
   S.img = imageData;
   S.w = canvas.width;
   S.h = canvas.height;
   S.origCanvas = canvas;
   ```
7. **Set default selected block** — center block of the image:
   ```js
   S.selBx = Math.floor(S.w / 16) * 8;
   S.selBy = Math.floor(S.h / 16) * 8;
   ```
8. **Dispatch custom event** `'imageReady'` on `document` so other sections can initialize

### Post-Upload UI

After successful upload:
- Upload zone fades out
- The uploaded image appears as a card (white border, shadow, `var(--radius)`)
- Below the image: `"Scroll down to begin ↓"` prompt with a subtle bounce animation
- Show image dimensions: `"Your photo: 480 × 360 pixels"`

---

## 3. Error Handling

- If file is not an image: show inline error message in the upload zone
- If image fails to load: show error message
- Style: red text, subtle red border on upload zone

---

## 4. Re-upload

- After upload, show a small "Change photo" link/button below the image
- Clicking it re-shows the upload zone (or triggers the file input again)

---

## 5. Privacy Note

- Below the upload zone: `"Your photo never leaves your device. All processing happens in your browser."`
- Style: `var(--text-muted)`, small font size (`0.85rem`)

---

## 6. Scroll Cue

After upload, add an animated scroll indicator:
- Down arrow or chevron
- Subtle bounce animation (CSS keyframes, `translateY(0) → translateY(8px) → translateY(0)`, `2s infinite`)
- Fades out once user scrolls past the hero section

---

## Acceptance Criteria

- [ ] Upload zone is visible, centered, and clickable
- [ ] Drag-and-drop works for image files
- [ ] Non-image files show an error
- [ ] Image is scaled to fit 480×480 and padded to 8×8
- [ ] `S.img`, `S.w`, `S.h`, `S.origCanvas` are populated after upload
- [ ] `imageReady` event fires after processing
- [ ] Post-upload UI shows the image with dimensions
- [ ] "Change photo" allows re-uploading
- [ ] Scroll cue animates below the image
- [ ] Layout is responsive — works on mobile screens
- [ ] Upload zone has proper hover and drag-over states
