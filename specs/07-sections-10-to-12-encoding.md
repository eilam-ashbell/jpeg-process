# Phase 07 — Sections 10–12: Encoding

**Goal:** Build the encoding pipeline sections: Zigzag Scanning, Run-Length Encoding, and Huffman Coding. These transform the quantized coefficients into the final compressed bitstream.

**Depends on:** Phase 01 (foundation), Phase 06 (quantized coefficients in `S.quantC`), Phase 03 (Data Size Bar)

---

## Section 10 — Zigzag Scanning (`#s-zigzag`)

### Purpose
Show how the 2D 8×8 matrix is read in zigzag order to cluster zeros together at the end.

### Text Content
Paragraph 1: The quantized 8×8 matrix still has two dimensions, but the compressed file is a one-dimensional stream of bytes. JPEG reads the matrix in a zigzag pattern — starting at the top-left corner and sweeping diagonally back and forth to the bottom-right.

Paragraph 2: Why zigzag? After quantization, the non-zero values cluster in the top-left (low frequencies) and zeros fill the bottom-right (high frequencies). The zigzag path puts all those zeros at the end of the sequence, making them easy to compress in the next step.

### Demo — Animated Zigzag Scan

```
┌──────────────────────────────────────┐
│  ┌──────────┐                        │
│  │ Quantized│  ← cursor traces path  │
│  │ matrix   │                        │
│  │ (8×8)    │                        │
│  └──────────┘                        │
│                                      │
│  [ ▶ Play Scan ]                     │
│                                      │
│  1D sequence:                        │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┐  │
│  │40 │-2 │ 3 │ 0 │-1 │ 0 │ 0 │...│  │
│  └───┴───┴───┴───┴───┴───┴───┴───┘  │
│                                      │
│  Non-zero values: 12                 │
│  Trailing zeros: 47                  │
└──────────────────────────────────────┘
```

**Implementation:**

1. **Quantized matrix display:**
   - 8×8 grid showing `S.quantC` values
   - Same matrix styling as Phase 06

2. **Zigzag lookup table:**
```js
const ZIGZAG = [
   0, 1, 8,16, 9, 2, 3,10,
  17,24,32,25,18,11, 4, 5,
  12,19,26,33,40,48,41,34,
  27,20,13, 6, 7,14,21,28,
  35,42,49,56,57,50,43,36,
  29,22,15,23,30,37,44,51,
  58,59,52,45,38,31,39,46,
  53,60,61,54,47,55,62,63
];
```

3. **Play Scan animation:**
   - Button triggers step-by-step animation
   - A highlighted cursor (accent blue outline) moves through the matrix in zigzag order
   - Each step: ~80ms interval
   - The cursor leaves a fading trail showing the path
   - As each cell is visited, its value appears in the 1D sequence below
   - Cells already visited get a subtle tint

4. **1D sequence display:**
   - Horizontal row of cells (scrollable if needed)
   - Each cell shows the coefficient value
   - Non-zero values: bold, accent-colored background
   - Zero values: dimmed, gray
   - As values appear during animation, they slide in from the left with a subtle animation

5. **Stats:**
   - Count of non-zero values
   - Count of trailing zeros (zeros after the last non-zero value — these can be dropped entirely)

**Zigzag scan:**
```js
function zigzagScan(quantized) {
  return ZIGZAG.map(i => quantized[i]);
}
```

Store: `S.zzSeq = zigzagScan(S.quantC);`

### Data Size Bar
- Stage: "After zigzag scan"
- Size: same as after quantization (~2–4 MB) — scan is just reordering

---

## Section 11 — Run-Length Encoding (RLE) (`#s-rle`)

### Purpose
Show how consecutive zeros are compressed into compact pair notation.

### Text Content
Paragraph 1: Now that all those zeros are clustered at the end of the sequence, JPEG uses a lossless trick called Run-Length Encoding. Instead of storing each zero individually, it counts consecutive zeros and writes a single pair: (number of zeros, next non-zero value). Runs of 15+ zeros are written as (15, 0).

Paragraph 2: The sequence ends with a special EOB (End of Block) marker that says "everything after this is zero." For a heavily quantized block, this marker might appear after just a handful of values — compressing 64 numbers down to maybe 10 or 15 tokens.

### Demo — Before/After Token Sequence

```
┌──────────────────────────────────────┐
│  Before RLE (zigzag sequence):       │
│  ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐    │
│  │40│-2│ 3│ 0│ 0│-1│ 0│ 0│ 0│ 0│... │
│  └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘    │
│                                      │
│  [ ▶ Apply RLE ]                     │
│                                      │
│  After RLE:                          │
│  ┌──────┬──────┬──────┬──────┬───┐   │
│  │(0,40)│(0,-2)│(0, 3)│(2,-1)│EOB│   │
│  └──────┴──────┴──────┴──────┴───┘   │
│                                      │
│  Tokens before: 64                   │
│  Tokens after: 5                     │
│  Reduction: 92%                      │
└──────────────────────────────────────┘
```

**Implementation:**

1. **Before sequence:**
   - Display `S.zzSeq` as a horizontal row of cells
   - Scrollable horizontally if needed
   - Zero cells: gray/dimmed; non-zero: bold

2. **Apply RLE button:**
   - Triggers animation of zero runs collapsing
   - Animation: groups of consecutive zeros **visually compress** together (spring animation — cells slide toward each other and merge)
   - Non-zero values stay in place; zero groups shrink into pair tokens
   - Duration: ~2s total, staggered per group
   - After animation: the "after" row shows the final RLE tokens

3. **After sequence:**
   - Each token displayed as a pair: `(skip, value)` where `skip` = number of preceding zeros
   - Special `EOB` token at the end (styled distinctly, accent blue background)
   - Tokens styled as small cards with rounded corners

4. **Stats:**
   - Token count before and after
   - Percentage reduction

**RLE Algorithm:**
```js
function rleEncode(zigzagSeq) {
  const tokens = [];
  let zeroCount = 0;

  // Find last non-zero
  let lastNonZero = zigzagSeq.length - 1;
  while (lastNonZero > 0 && zigzagSeq[lastNonZero] === 0) lastNonZero--;

  for (let i = 0; i <= lastNonZero; i++) {
    if (zigzagSeq[i] === 0) {
      zeroCount++;
      if (zeroCount === 16) {
        tokens.push({ skip: 15, value: 0 }); // ZRL
        zeroCount = 0;
      }
    } else {
      tokens.push({ skip: zeroCount, value: zigzagSeq[i] });
      zeroCount = 0;
    }
  }
  tokens.push({ skip: 0, value: 'EOB' }); // End of block
  return tokens;
}
```

Store: `S.rleSeq = rleEncode(S.zzSeq);`

### Data Size Bar
- Stage: "After RLE"
- Size: ~800 KB
- Color: Red

---

## Section 12 — Huffman Coding (`#s-huffman`)

### Purpose
Show how Huffman coding assigns shorter binary codes to more frequent values, achieving the final compression.

### Text Content
Paragraph 1: The last step is Huffman coding — a lossless technique that replaces each RLE token with a binary code. Frequent values (like "zero" and small coefficients) get very short codes (maybe 2–3 bits), while rare values get longer codes. It's the same principle that makes Morse code efficient — "E" is just a single dot.

Paragraph 2: The result is a stream of bits that's as compact as possible without losing any information. Combined with all the previous steps, your original 36 MB of raw pixel data is now compressed to around 300 KB — over 100 times smaller. That's the magic of JPEG.

### Demo — Huffman Tree + Code Table

```
┌──────────────────────────────────────┐
│  ┌────────────────────────────────┐  │
│  │                                │  │
│  │  [Huffman tree visualization]  │  │  ← Canvas/SVG
│  │                                │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │ Symbol │ Freq │ Code │ Bits   │  │
│  │   0    │  45  │  0   │  1     │  │  ← Code table
│  │  -1    │   8  │  10  │  2     │  │
│  │   1    │   6  │  110 │  3     │  │
│  │  -2    │   3  │  1110│  4     │  │
│  │  ...   │  ... │ ...  │  ...   │  │
│  └────────────────────────────────┘  │
│                                      │
│  Total bits: 2,847                   │
│  Average bits/symbol: 3.2            │
│  vs. fixed 8 bits: 60% savings       │
└──────────────────────────────────────┘
```

**Implementation:**

1. **Huffman Tree Construction:**
   - Count frequency of each unique value in the RLE token stream
   - Build the Huffman tree (standard algorithm with priority queue)
   - Generate binary codes by traversing the tree

```js
function buildHuffmanTree(tokens) {
  // Count frequencies
  const freq = {};
  tokens.forEach(t => {
    const key = `${t.skip},${t.value}`;
    freq[key] = (freq[key] || 0) + 1;
  });

  // Build tree using a min-heap
  const nodes = Object.entries(freq).map(([symbol, count]) => ({
    symbol, count, left: null, right: null
  }));

  // Sort-based priority queue (simple for small sets)
  while (nodes.length > 1) {
    nodes.sort((a, b) => a.count - b.count);
    const left = nodes.shift();
    const right = nodes.shift();
    nodes.push({
      symbol: null,
      count: left.count + right.count,
      left, right
    });
  }

  return nodes[0]; // root
}

function generateCodes(node, prefix = '', codes = {}) {
  if (!node) return codes;
  if (node.symbol !== null) {
    codes[node.symbol] = prefix || '0';
    return codes;
  }
  generateCodes(node.left, prefix + '0', codes);
  generateCodes(node.right, prefix + '1', codes);
  return codes;
}
```

2. **Tree Visualization (2D — Canvas or SVG):**
   - Draw the Huffman tree as a top-down node graph
   - Root at top, leaves at bottom
   - Leaf nodes: circles with the symbol value, colored blue (`var(--accent-blue)`)
   - Internal nodes: smaller gray circles
   - Edges: lines with "0" (left) and "1" (right) labels
   - Hover over any leaf: highlight the path from root, show the full binary code
   - Tree auto-layouts using a recursive positioning algorithm

   (Note: A 3D Three.js version replaces this in Phase 09)

3. **Code Table:**
   - HTML table with columns: Symbol, Frequency, Huffman Code, Bit Length
   - Sorted by frequency (most frequent first)
   - Rows alternate background for readability
   - Code column in monospace (JetBrains Mono)
   - Hovering a row highlights the corresponding leaf node in the tree

4. **Stats:**
   - Total bits needed with Huffman coding
   - Average bits per symbol
   - Comparison with fixed-length (8-bit) coding — show percentage savings

### Data Size Bar
- Stage: "After Huffman coding"
- Size: ~300 KB (final JPEG size)
- Color: Red
- Show compression ratio badge: e.g., "120× smaller"

---

## Shared Notes

### Event Listeners
All three sections listen for `blockChanged` and `qualityChanged` events:
- `blockChanged`: re-extract → re-DCT → re-quantize → re-zigzag → re-RLE → re-Huffman
- `qualityChanged`: re-quantize → re-zigzag → re-RLE → re-Huffman

### Animation Tokens
For token/sequence animations, use a shared animation utility:
```js
function animateSequence(elements, interval, onStep) {
  let i = 0;
  const timer = setInterval(() => {
    if (i >= elements.length) { clearInterval(timer); return; }
    onStep(elements[i], i);
    i++;
  }, interval);
  return timer;
}
```

### Scroll-Triggered Auto-Play
Consider auto-triggering the "Play Scan" (S10) and "Apply RLE" (S11) animations when the section scrolls into view for the first time, with a short delay. The button allows replay.

---

## Acceptance Criteria

- [ ] S10: Quantized matrix displays correctly
- [ ] S10: Zigzag lookup table produces correct order
- [ ] S10: Play animation traces the zigzag path with visual cursor
- [ ] S10: 1D sequence populates as animation plays
- [ ] S10: Stats show correct non-zero count and trailing zeros
- [ ] S11: Before sequence shows zigzag data
- [ ] S11: Apply RLE animation collapses zero runs visually
- [ ] S11: After sequence shows correct RLE pairs with EOB
- [ ] S11: Stats show token count reduction
- [ ] S12: Huffman tree builds correctly from RLE data
- [ ] S12: Tree visualization renders with correct structure
- [ ] S12: Code table shows correct codes sorted by frequency
- [ ] S12: Hovering tree leaf / table row cross-highlights
- [ ] S12: Stats show bit savings
- [ ] All sections update on `blockChanged` and `qualityChanged`
- [ ] Data Size Bar updates correctly for each section
- [ ] Mobile: sequences scroll horizontally, tree scales appropriately
