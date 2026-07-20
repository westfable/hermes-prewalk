# Frontend Verification Recipes

Two recipes for diagnosing responsive/overlay layout bugs that the default browser tooling can't reveal. Both were battle-tested on the ROGUE FM "noise block" investigation (July 2026), where the built-in screenshot's narrow viewport hid a desktop-only regression and code reading alone couldn't attribute a visual artifact to its element.

## Why the built-in screenshot isn't enough

The built-in `browser_vision` screenshot renders at a narrow (mobile-ish) viewport by default. A "desktop looks fine" shot can completely miss layout bugs that only appear at ≥1024px: bare empty regions, wrong overlay/z-index stacking, collapsed grid/flex tracks. For any responsive or desktop CSS work, verify at explicit desktop dimensions.

## Recipe 1: Playwright desktop-pinned screenshots

```js
// scripts/<feature>-screenshots/capture.mjs
import { chromium } from "playwright";

const browser = await chromium.launch();
const page = await browser.newPage({ viewport: { width: 1600, height: 900 } });
await page.goto("http://localhost:3000/<route>", { waitUntil: "networkidle" });
await page.screenshot({ path: "scripts/<feature>-screenshots/after-desktop.png", fullPage: true });

// Optionally also capture mobile for comparison
await page.setViewportSize({ width: 390, height: 844 });
await page.screenshot({ path: "scripts/<feature>-screenshots/after-mobile.png", fullPage: true });

await browser.close();
```

Run with `node scripts/<feature>-screenshots/capture.mjs` (Playwright must be installed: `npm i -D playwright && npx playwright install chromium`).

Conventions:
- Capture **before** shots prior to editing, **after** shots when done — keep both in `scripts/<feature>-screenshots/` for visual diff
- Pin the viewport explicitly (1600×900 is a good desktop default); never rely on the tool's default
- `fullPage: true` catches below-the-fold regressions

## Recipe 2: DOM geometry probe (which element owns this pixel?)

When a region looks wrong but screenshots are ambiguous — "what is that block in the corner?" — ask the DOM directly instead of guessing from code:

```js
// In the same Playwright script, or via browser_console on the live page:
const probe = await page.evaluate(() => {
  const [x, y] = [1400, 300]; // the suspicious pixel
  const stack = document.elementsFromPoint(x, y).map(el => ({
    tag: el.tagName,
    cls: el.className?.toString?.().slice(0, 80),
    rect: el.getBoundingClientRect().toJSON(),
    z: getComputedStyle(el).zIndex,
    pos: getComputedStyle(el).position,
    opacity: getComputedStyle(el).opacity,
  }));
  return stack;
});
console.log(JSON.stringify(probe, null, 2));
```

Interpretation:
- The **first** element in the stack is what the user actually sees at that pixel
- Compare its `rect` to the visual artifact's bounds — a match confirms attribution
- `z-index` + `position` explain why it's winning the stacking contest
- Decorative layers (grain overlays, shader rails, scanlines) often sit at low opacity over everything — probe several to find which one produces the artifact

This is how the ROGUE FM "noise block" root cause — an art container at `lg:left-[28%]` leaving a bare region — was confirmed after two rounds of geometry probing, when code reading alone could not pick the culprit among four overlapping decorative layers.

## When to use which

| Symptom | Recipe |
|---|---|
| "Desktop layout looks broken but my screenshot looks fine" | 1 — pin the viewport |
| "There's a visual artifact and I don't know which element causes it" | 2 — elementsFromPoint probe |
| "Overlay/z-index fight between decorative layers" | 2, probing each layer's computed style |
| Before/after evidence for a CSS refactor | 1, both viewports, kept in scripts/ |

## Verification cadence reminder

- CSS-only changes: `npx tsc --noEmit` + a pinned-viewport screenshot is sufficient
- Skip `npm run build` for visual polish — slow, and unrelated pre-existing route errors surface as false failures
- Full build belongs at commit time, not every iteration
