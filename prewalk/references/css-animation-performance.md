# CSS Animation Performance: Diagnosing Scroll Jank

Use when a task reports choppy scroll, janky/glitchy feel, or "loads slow" on a page or section — a *performance* complaint, not a layout defect. Prewalk pitfall #12's visual pass finds what looks wrong; this reference finds what's **expensive**.

## Step 1 — Enumerate animated layers (browser_console)

Run against the section root to list every element with an active animation:

```js
var all=document.querySelectorAll('.SECTION-CLASS *');var out=[];
all.forEach(function(el){var a=getComputedStyle(el).animationName;
if(a&&a!=='none')out.push({cls:el.className.slice(0,60),anim:a});});
JSON.stringify(out)
```

Any element with an **infinite** animation is a per-frame cost for as long as it's mounted — even when scrolled off-screen.

## Step 2 — Classify by animated property

The keyframes' *property* determines cost, not the element's size:

| Keyframes animate | Cost class | Notes |
|---|---|---|
| `transform`, `opacity` | Composited (GPU) | Cheap — safe to keep |
| `filter`, `clip-path` | Sometimes composited | Browser-dependent |
| `text-shadow`, `box-shadow` | **Repaint every frame** | Blur radius = re-rasterize; worse on large text |
| `background-position`, `background-size` | **Repaint every frame** | A 4px scanline tile still repaints the whole element box |
| `width/height/top/left/margin` | Layout + repaint | Worst — never animate |

Caveat: an FPS probe (`requestAnimationFrame` count over 2s) on a desktop/headless browser can read 144fps while real devices still jank — GPU headroom hides the cost. Trust the property classification over the FPS number.

## Fix patterns (minimal-intervention — keep the look)

### A. Flickering shadow glow (text-shadow / box-shadow)
- Delete the infinite keyframes; use a **static** shadow with fewer layers and smaller blur.
- Add `will-change: transform; transform: translateZ(0)` so the element composites during scroll.
- If the flicker is part of the identity, move it to `:hover` gated by `@media (hover: hover)` — user-initiated, zero scroll cost.

### B. Drifting background pattern (scanlines, grain)
Move the pattern to a `::before` pseudo-element and animate `transform: translateY(N)`, where N = drift distance (must be a multiple of the tile height). Give the pseudo `inset: -Npx 0 0 0` so the loop wraps seamlessly; parent gets `overflow: hidden`.

**Two pitfalls that silently keep the bug or add a new one:**
1. Update the `@keyframes` to animate `transform` — converting the application site but leaving the keyframes on `background-position` changes nothing.
2. Move the background **entirely** to the pseudo-element. A static copy left on the parent doubles the overlay density, and the two layers drift relative to each other producing a moiré/shimmer. Parent keeps only position/opacity/overflow.

### C. Section isolation
`contain: layout style paint;` on the section so absolute-positioned decorative children don't expand the paint area during scroll.

### D. Mobile
- Decorative infinite animations: `display: none` below 768px — mobile GPUs gain the most.
- Large display text: shrink the `clamp()` floor/ceiling; raster cost scales with glyph area.

### E. Reduced motion (always)
```css
@media (prefers-reduced-motion: reduce) {
  .animated-thing, .animated-thing::before { animation: none !important; }
}
```

## Verify
- Re-run the enumeration snippet — remaining animations should be transform/opacity only.
- CSS-only change → `npx tsc --noEmit` is sufficient (prewalk pitfall #10); skip `npm run build`.
- `browser_vision` before/after screenshot to confirm the look survived.

## Worked example (ROGUE FM mission hook, `/` homepage)
Report: "glitching/flashing red text makes scroll choppy on desktop and mobile." Enumeration found exactly two infinite repaint animations:
- `.v3-mission-hook-dek` — `v3-vhs-red-flicker` 3.2s on a 5-layer `text-shadow` (blur up to 26px) at `clamp(2.125rem, 7.25vw, 4.25rem)` → replaced with a 2-layer static glow + hover pulse + `will-change`.
- `.v3-mission-hook-scanlines` — `scanlines-drift` 8s on `background-position` over a full-section overlay → moved to `::before` with `translateY(64px)` (64 = 16× the 4px tile), `contain: layout style paint` on the section, scanlines `display: none` on mobile, dek clamp reduced to `clamp(1.5rem, 5.5vw, 2rem)` under 768px.
