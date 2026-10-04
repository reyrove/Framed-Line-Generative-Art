# Framed-Line-Generative-Art

A seed-based generative art catalogue for **Framed Line**, a computational gesture system. Built as a single-file HTML page with p5.js, styled as a gallery-ready portfolio and licensing surface.

**Live:** `https://reyrove.github.io/Framed-Line-Generative-Art/`

---

## Overview

**Framed Line** renders a single wandering gesture from numeric seeds. Each composition places 6–14 points at random positions inside a bordered ground, then connects them in sequence with curvy quadratic Bézier segments. Each segment has its own hue along a gradient, and its own thickness along a second gradient. A colourful frame surrounds the composition, and small scattered echoes add texture near the vertices.

The page functions as a **catalogue and licensing surface** for fashion houses, textile studios, and surface designers. Every composition is reproducible by seed, meaning the same number always produces the identical gesture.

---

## Features

### Generative Engine (p5.js)

- **6–14 points** — placed at random positions inside the frame, with 5% padding from the canvas edge
- **Quadratic Bézier curve** — consecutive points connected by a curve whose control point is the midpoint, offset by a random displacement (±5% of canvas)
- **Gradient hue** — the line shifts colour smoothly from a random start hue to a random end hue along the path
- **Varying thickness** — stroke weight interpolates from `cs × 0.001` (hairline) to `cs × 0.01` (heavy)
- **Colourful frame** — a thick border at 5% padding, drawn in a random hue at 80% saturation and 90% brightness
- **Texture echoes** — a handful of tiny points with 40% alpha, scattered near each vertex

### Same Seed → Many Surfaces

A single seed drives every instance on the page:

| Instance | Purpose | Tiling |
|----------|---------|--------|
| **Cover** | Hero composition | 1 × 1 |
| **Plate** | Main work (master) | 1 × 1 |
| **Frame** | Framed archival print | 1 × 1 |
| **Print** | Archival print surface | 1 × 1 |
| **Scarf** | Silk scarf (3:1) | 3 × 1 |
| **Textile** | Fabric yardage (4:3) | 2 × 2 |
| **Wall** | Wallpaper (2:3) | 1 × 2 |
| **Archive** | 8 unique seeds | 1 × 1 each |

### Controls

- **New Seed** button / `R` key — regenerate with a fresh seed
- **Download** button / `S` key — export the plate canvas as PNG
- **Click any archive plate** — load that seed into the main view

### Performance

- Debounced resize handling (250ms)
- `IntersectionObserver` pauses off-screen sketches
- `visibilitychange` pauses rendering when the tab is hidden
- `pixelDensity(1)` for a lighter canvas footprint
- Two-frame render (`maxFrames: 2`) so tiles draw once and stop

---

## File Structure

```
Framed-Line-Generative-Art/
├── index.html          # Single-file page (structure + styles + generative engine)
├── README.md
└── images/
    ├── fav.svg         # Favicon
    ├── tote.png        # Mockup: tote bag
    ├── tee.png         # Mockup: t-shirt
    └── cushion.png     # Mockup: cushion
```

No build step. No dependencies to install. p5.js is loaded from a CDN.

---

## Getting Started

1. Clone the repo:
   ```bash
   git clone https://github.com/reyrove/Framed-Line-Generative-Art.git
   cd Framed-Line-Generative-Art
   ```
2. Add the three mockup images to `images/` (optional — they appear in the Commission section).
3. Open `index.html` in a browser.

That's it. The page self-initialises on `DOMContentLoaded`.

---

## How the Generative System Works

### Seeded Random

A linear congruential generator replaces p5's native randomness inside each sketch:

```js
class SeededRandom {
  constructor(s) { this.state = (Math.abs(s) % 2147483647) || 1; }
  rand() { this.state = (this.state * 9301 + 49297) % 233280; return this.state / 233280; }
  range(min, max) { return this.rand() * (max - min) + min; }
  int(min, max) { return Math.floor(this.range(min, max + 1)); }
  pick(arr) { return arr[Math.floor(this.rand() * arr.length)]; }
}
```

Every aesthetic decision — frame hue, point count, point positions, gradient hues, echo placement — is drawn from this generator in a fixed order. Same seed, same sequence, same composition.

### Parameter Space (per seed)

| Parameter | Range |
|-----------|-------|
| Point count | 6 – 14 |
| Frame padding | 5% of canvas |
| Frame stroke weight | `cs × 0.02` |
| Frame hue | 0 – 360 |
| Line stroke weight | `cs × 0.001` → `cs × 0.01` (gradient) |
| Line start hue | 0 – 360 |
| Line end hue | 0 – 360 |
| Curve control offset | ±5% of canvas |
| Echo probability | 20% per vertex |

### The Gesture

The line is drawn segment by segment. Each segment connects two consecutive points with a quadratic Bézier curve:

```js
p.beginShape();
p.vertex(p1.x, p1.y);
const midX = (p1.x + p2.x) / 2 + sr.range(-cs * 0.05, cs * 0.05);
const midY = (p1.y + p2.y) / 2 + sr.range(-cs * 0.05, cs * 0.05);
p.quadraticVertex(midX, midY, p2.x, p2.y);
p.endShape();
```

The midpoint is offset by a random displacement, so each segment bulges slightly. The result is a meandering path that reads as a single continuous gesture without ever taking a straight line.

### Gradient Colour and Thickness

Both hue and stroke weight are interpolated along the path:

```js
const t = i / (points.length - 1);
const hue = hueStart + (hueEnd - hueStart) * t;
p.stroke(((hue % 360) + 360) % 360, 80, 90);
p.strokeWeight(cs * (0.001 + t * 0.009));
```

The line begins as a hairline in one hue, and ends as a heavy line in another. This gives the composition a natural sense of direction — it reads as a movement from one end to the other.

### Texture Echoes

After the main line, a handful of tiny points are scattered near the vertices:

```js
for (let i = 0; i < points.length; i++) {
  if (sr.rand() < 0.2) {
    p.stroke(sr.rand() * 360, 80, 90, 0.4);
    p.strokeWeight(cs * 0.002);
    p.point(points[i].x + sr.range(-cs * 0.02, cs * 0.02),
            points[i].y + sr.range(-cs * 0.02, cs * 0.02));
  }
}
```

These are subtle — 20% of vertices, 40% alpha, tiny — and add a light dusting of secondary marks without competing with the main line.

### Tiling Logic

Rectangular surfaces (scarf, textile, wall) tile the square composition N×M. Each tile is drawn into its own translated coordinate space, centred within its cell:

```js
const tileW = canvasW / tileCols;
const tileH = canvasH / tileRows;
const tileSize = Math.min(tileW, tileH);
// draw square of side tileSize at (ox, oy) for each cell
```

This preserves the square composition's proportions regardless of surface aspect ratio.

---

## Customisation

### Change the default seed

Edit the init function:

```js
function init() {
  seed = 123456789;  // fixed seed instead of random
  // ...
}
```

### Adjust the point count range

```js
const totalPoints = sr.int(6, 14);  // change to sr.int(4, 20) for fewer / more points
```

Fewer points produce a sweeping, minimal gesture. More points produce a busier, more detailed path.

### Adjust the curve displacement

```js
const midX = (p1.x + p2.x) / 2 + sr.range(-cs * 0.05, cs * 0.05);
```

Change `0.05` to a smaller value for straighter segments, or a larger value for wilder curves.

### Adjust the gradient behaviour

The gradient is currently linear from `hueStart` to `hueEnd`. To reset the original sketch's behaviour (each segment a fresh random hue):

```js
// Replace:
const hue = hueStart + (hueEnd - hueStart) * t;
// With:
const hue = sr.rand() * 360;
```

### Adjust the frame

```js
p.strokeWeight(cs * 0.02);
p.stroke(sr.rand() * 360, 80, 90);
p.rect(framePadding / 2, framePadding / 2, cs - framePadding, cs - framePadding);
```

- Change the `0.02` multiplier for a thinner or thicker frame.
- Replace `sr.rand() * 360` with a fixed hue for a consistent frame across seeds.
- Change `framePadding = cs * 0.05` to move the frame closer to or further from the edge.

### Add texture echoes

The current probability is 20%. To add more:

```js
if (sr.rand() < 0.2) { ... }  // change 0.2 to 0.4 or 0.6
```

### Change surface tiling

Edit `surfaceDefs`:

```js
const surfaceDefs = [
  { id: 'surfacePrint',   wrapId: 'wrap-surfacePrint',   tile: { cols: 1, rows: 1 } },
  { id: 'surfaceScarf',   wrapId: 'wrap-surfaceScarf',   tile: { cols: 3, rows: 1 } },
  // ...
];
```

The `tile` object accepts any `{ cols, rows }` pair.

### Replace mockup images

Swap the files in `images/` or update the `src` attributes in the Commission section:

```html
<img src="images/your-mockup.png" alt="..." loading="lazy" />
```

---

## Design System

| Token | Value | Use |
|-------|-------|-----|
| `--bg` | `#0a0a0a` | Page background |
| `--paper` | `#f4f1ea` | Light surface (Surfaces section) |
| `--gold` | `#b8935a` | Primary accent |
| `--gold-soft` | `#d4b483` | Italic emphasis |
| `--serif` | Cormorant Garamond | Headings, titles |
| `--mono` | DM Mono | Labels, metadata, UI |

Type scale, spacing, and section rhythm match the Skyline, Portrait, Girih, Gothic Grid, Order to Chaos, Wallpaper Groups, Wandering Tiles, and Food Grid templates. Responsive breakpoints at 900px, 720px, 560px, and 400px.

---

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `R` | Regenerate with a new seed |
| `S` | Download plate as PNG |
| `Enter` / `Space` | Activate focused archive item |
| `Esc` | Close legal modal |

---

## Browser Support

Modern evergreen browsers with:

- Canvas 2D (universal)
- `IntersectionObserver` (universal in modern browsers)
- `backdrop-filter` (Safari 9+, Chrome 76+, Firefox 103+)
- CSS custom properties (universal)

Fallback: if `IntersectionObserver` is unavailable, sketches render continuously.

---

## Accessibility

- Archive items are keyboard-focusable with `role="button"` and `aria-label`
- Legal modal traps focus, closes on `Esc` and overlay click
- `prefers-reduced-motion` disables transitions and smooth scroll
- All interactive elements have visible focus states (via browser default)

---

## Section Map

| Section | ID | Purpose |
|---------|-----|---------|
| Cover | — | Hero title + live gesture |
| Statement | `#statement` | Artist statement |
| Plate | `#work` | Main work + metadata + controls |
| Surfaces | `#application` | Framed print + 4 tiled surfaces |
| Process | `#process` | Three process cards |
| Archive | `#archive` | 8 selectable seeds |
| Commission | `#commission` | Services + mockups + CTA |
| Colophon | — | Footer, links, legal |
| Legal modal | `#legalModal` | Licensing / Terms / Credits |

---

## Deployment (GitHub Pages)

1. Push the repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Source**, select `Deploy from a branch`.
4. Choose branch `main` and folder `/ (root)`.
5. Save — the site goes live at `https://reyrove.github.io/Framed-Line-Generative-Art/`.

---

## Notes on the Source Sketch

The generative engine is adapted almost verbatim from the original p5 sketch. The adaptations that turn it into a catalogue page:

1. **Seeded random replaces p5's `random()`** — a local `SeededRandom` class drives every choice. The original sketch calls `setSeeds()` for fxhash compatibility; that's replaced with the seeded generator directly.
2. **Sketch factory** — `makeSketch({ slotId, seedValue, ... })` returns a p5 sketch that can be instantiated into any DOM slot, enabling cover / plate / frame / surfaces / archive from one seed.
3. **DOM-driven canvas sizing** — `cs = min(windowHeight, windowWidth)` becomes slot-rect dimensions.
4. **Tiling support** — the `tile: { cols, rows }` parameter repeats the square composition across rectangular surfaces.
5. **Drawing logic takes `p` explicitly** — the original used p5's global mode; here `p` is passed so the factory closure works.
6. **Gradient hue fixed** — the original sketch calls `random(0, 360)` twice inside `map(...)` for every segment, which means each segment gets a fresh random hue (no gradient). In this adaptation, the start and end hues are drawn once and interpolated linearly along the path — a true gradient. If you want the original behaviour, replace the `const hue = ...` line with `const hue = sr.rand() * 360`.
7. **`point()` calls use `sr.rand()`** — the tiny echoes are deterministic per seed.
8. **`windowResized` replaced by a debounced `handleResize`** — all sketches are destroyed and recreated on resize (250ms debounce), since their canvases are locked to slot dimensions.
9. **`triggerPreview` removed** — the original calls it for fxhash thumbnail generation; not needed in the plain-browser catalogue.

---

## Related Repositories

| Repo | Theme |
|------|-------|
| [`Skyline-Series`](https://github.com/reyrove/Skyline-Series) | Themed cityscapes |
| [`Portrait-Series-Generative-Art`](https://github.com/reyrove/Portrait-Series-Generative-Art) | Generative faces |
| [`Girih-2-Generative-Art`](https://github.com/reyrove/Girih-2-Generative-Art) | Islamic geometric ornament |
| [`Gothic-Grid-Generative-Art`](https://github.com/reyrove/Gothic-Grid-Generative-Art) | Mosaic grid composition |
| [`Order-to-Chaos-Generative-Art`](https://github.com/reyrove/Order-to-Chaos-Generative-Art) | Gradient grid composition |
| [`Wallpaper-Groups-Generative-Art`](https://github.com/reyrove/Wallpaper-Groups-Generative-Art) | Crystallographic symmetry |
| [`Wandering-Tiles-Generative-Art`](https://github.com/reyrove/Wandering-Tiles-Generative-Art) | Line-drawing composition |
| [`Food-Grid-Generative-Art`](https://github.com/reyrove/Food-Grid-Generative-Art) | Illustrated food grid |
| **`Framed-Line-Generative-Art`** | Gesture line composition *(this repo)* |

All nine share the same catalogue template, design tokens, and seed-based reproducibility model.

---

## Credits

- **Design & Generative System** — Reyhaneh Daneshdoost
- **Typefaces** — Cormorant Garamond, DM Mono
- **Engine** — p5.js 1.9.0
- **Edition** — Framed Line, Autumn 2026

---

## License

All artwork, code, and generative systems are the intellectual property of the artist. Compositions may not be reproduced, redistributed, resold, or adapted without a written license.

For licensing, commissions, or private systems: **reyhanehdaneshdoost@gmail.com**