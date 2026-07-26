# Design Rules — verified implementation (2026)

Copy-paste CSS / Tailwind / JS for the three mandatory global rules. Browser-support claims are
verified (MDN / caniuse / spec, July 2026). Wire these once at the token/base layer so they hold
everywhere automatically.

## Browser-support matrix (load-bearing facts)

| Feature | Status | Ship now? |
|---|---|---|
| `contrast-color()` | Baseline **newly available (Apr 2026)**; Chrome/Edge 147, Firefox 146, Safari 26 (~76%) | Yes, **with fallback** |
| `color-contrast()` (old name) | Never shipped stable — redesigned/renamed to `contrast-color()` | **No — never use** |
| `light-dark()` | Baseline 2024 (needs `color-scheme: light dark`) | Yes |
| Relative color `rgb(from …)` / `oklch(from …)` | Baseline (~88%); Chrome 131, Safari 18, Firefox 133 | Yes |
| `color-mix()` | Baseline 2023 | Yes |

Target `contrast-color()`, never `color-contrast()`.

---

## RULE A — Auto-contrast font color (guarantee legibility on any background)

### Why literal RGB inversion (`255−r,255−g,255−b`) FAILS
Inversion mirrors a color through the RGB cube's center, so any **mid-tone** lands right next to its
inverse — near-zero contrast. `#808080 → #7F7F7F`: both luminance ≈ 0.21, contrast ratio ≈ **1.02:1**
(WCAG AA needs 4.5:1 body / 3:1 large). Inversion also flips hue (blue→orange). It optimizes color
*distance*, not *luminance* contrast — and only luminance contrast is legibility. So "invert to the
opposite" is honored as **maximize contrast against the background**.

### The correct algorithm
1. **Linearize sRGB** for each channel `c8` (0–255): `cs = c8/255; c_lin = cs≤0.04045 ? cs/12.92 : ((cs+0.055)/1.055)^2.4`
2. **Relative luminance:** `L = 0.2126·R_lin + 0.7152·G_lin + 0.0722·B_lin`
3. **Contrast ratio:** `CR = (L_light + 0.05) / (L_dark + 0.05)` (1:1 … 21:1)
4. **Pick max-contrast text:** compare CR(bg,white) vs CR(bg,black), take larger. Crossover is
   **L\* = 0.179** (not 0.5): `bg luminance > 0.179 → dark text, else light text`.

Use tuned endpoints `#fafafa` / `#0a0a0a` (softer than pure #fff/#000, still ~19:1) — but then compute
both CRs explicitly rather than using the 0.179 fast-path.

### Solution 1 — pure CSS, layered fallback (components set only `--bg`)
```css
.surface {
  --bg: #4f46e5;                 /* the ONLY thing a component sets */
  background-color: var(--bg);
  color: #fff;                   /* Tier 1: static fallback (oldest browsers) */
}
/* Tier 2 — relative color (Baseline 2024): snap to B/W by OKLCH lightness */
@supports (color: oklch(from red l c h)) {
  .surface {
    --threshold: 0.62;
    color: oklch(from var(--bg) clamp(0, (var(--threshold) - l) * infinity, 1) 0 h);
  }
}
/* Tier 3 — the real deal (Baseline 2026): browser guarantees a WCAG-AA B/W choice */
@supports (color: contrast-color(black)) {
  .surface { color: contrast-color(var(--bg)); }
}
```
`@supports` cascades bottom-up: modern browsers use `contrast-color()`, 2024-25 use the OKLCH trick,
ancient ones the static fallback. **Contract: components only ever set `--bg`; `--fg` is derived, so
no token can ever produce invisible text.**

### Solution 2 — JS/TS utility (SSR-safe)
```ts
type RGB = [number, number, number];
const srgbToLinear = (c: number) => { const cs = c/255; return cs <= 0.04045 ? cs/12.92 : Math.pow((cs+0.055)/1.055, 2.4); };
const luminance = ([r,g,b]: RGB) => 0.2126*srgbToLinear(r) + 0.7152*srgbToLinear(g) + 0.0722*srgbToLinear(b);
const contrast = (a: number, b: number) => { const [hi,lo] = a>=b ? [a,b] : [b,a]; return (hi+0.05)/(lo+0.05); };
function parseHex(hex: string): RGB { let h = hex.replace('#','').trim(); if (h.length===3) h = h.split('').map(c=>c+c).join(''); const n = parseInt(h,16); return [(n>>16)&255,(n>>8)&255,n&255]; }
export function bestTextColor(bg: string, { light='#fafafa', dark='#0a0a0a' } = {}): string {
  const L = luminance(parseHex(bg));
  return contrast(L, luminance(parseHex(dark))) >= contrast(L, luminance(parseHex(light))) ? dark : light;
}
export function passesAA(bg: string, fg: string, large = false) {
  return contrast(luminance(parseHex(bg)), luminance(parseHex(fg))) >= (large ? 3 : 4.5);
}
```
Usage: `el.style.setProperty('--fg', bestTextColor(bg))`. (Browser-only variant can parse any CSS color
via a `<canvas>` 1×1 `getImageData`; composite alpha over the page bg before measuring.)

### Text over IMAGES / GRADIENTS (per-pixel bg unknown → force a known local bg)
```css
/* (a) Scrim — gradient veil under the text band. Best for hero headings. */
.on-image { position: relative; color: #fff; }
.on-image::before { content:""; position:absolute; inset:0; pointer-events:none;
  background: linear-gradient(0deg, rgba(0,0,0,.65), rgba(0,0,0,.25) 45%, transparent); }
.on-image > * { position: relative; }
/* (b) Text-shadow halo — captions over busy images */
.on-image-shadow { color:#fff; text-shadow: 0 1px 2px rgba(0,0,0,.9), 0 0 6px rgba(0,0,0,.6); }
/* (c) Backdrop chip — badges/labels over photography */
.on-image-chip { color:#fff; padding:.2em .5em; border-radius: var(--radius-md,.75rem);
  background: color-mix(in srgb, black 35%, transparent); backdrop-filter: blur(8px); -webkit-backdrop-filter: blur(8px); }
```
Never use `bestTextColor()` for images — there is no single bg to read. Use a scrim/shadow/chip.

---

## RULE B — 3D hover triple-invert on every clickable
Border + background + text invert **together** (bg↔fg swap) plus a real 3D lift; full a11y + reduced-motion.
```css
.btn3d {
  --bg: #4f46e5; --fg: #fff; --lift: 6px;
  display:inline-flex; align-items:center; gap:.5em;
  padding:.7rem 1.15rem; font:inherit; font-weight:600; line-height:1; cursor:pointer;
  border:2px solid var(--bg); border-radius: var(--radius-md,.75rem);   /* Rule C */
  background-color: var(--bg); color: var(--fg);
  transform: translateY(0) scale(1); box-shadow: 0 1px 2px rgba(0,0,0,.12);
  transition: transform .18s cubic-bezier(.2,.8,.2,1), box-shadow .18s, background-color .18s, color .18s, border-color .18s;
  will-change: transform;
}
.btn3d:hover {                    /* triple invert + lift */
  background-color: var(--fg); color: var(--bg); border-color: var(--bg);
  transform: translateY(calc(var(--lift) * -1)) scale(1.02);
  box-shadow: 0 calc(var(--lift) * .5) 0 -1px color-mix(in srgb, var(--bg) 45%, transparent),
              0 calc(var(--lift) * 2) calc(var(--lift) * 2.5) rgba(0,0,0,.28);
}
.btn3d:active { transform: translateY(-1px) scale(.99); box-shadow: 0 1px 2px rgba(0,0,0,.25); }
.btn3d:focus-visible { outline:none; box-shadow: 0 0 0 3px color-mix(in srgb, var(--bg) 45%, transparent), 0 6px 14px rgba(0,0,0,.2); }
@media (prefers-reduced-motion: reduce) {
  .btn3d { transition-property: background-color, color, border-color, box-shadow; }
  .btn3d:hover, .btn3d:active { transform: none; }   /* keep the color invert, drop the movement */
}
```
If you set `--fg: contrast-color(var(--bg))`, add `.btn3d:hover { color: contrast-color(var(--fg)); }` so the
swapped state stays legible. Prefer one `.btn3d` class over long arbitrary-variant chains.

---

## RULE C — Global minimum radius, never sharp edges
Literal `border-radius:10%` is size-relative → explodes on big cards, vanishes on chips. Use a **fixed
token scale with a hard floor** (10px / 0.625rem), optional bounded fluid `clamp()`.
```css
:root {
  --radius-min:  0.625rem;                          /* 10px hard floor */
  --radius-none: 0.625rem;                          /* remap "none" to the floor → sharp is unreachable */
  --radius-sm: .5rem; --radius-md: .75rem; --radius-lg: 1rem; --radius-xl: 1.5rem; --radius-2xl: 2rem;
  --radius-full: 9999px;                            /* pills / circles */
  --radius-fluid: clamp(0.625rem, 10%, 1.5rem);     /* proportional, bounded */
}
/* Zero-specificity global net: any real component rule still wins, but sharp corners can't render. */
:where(*):not(html, body, :root, hr, .sharp, [data-sharp]) { border-radius: max(var(--radius-min), var(--_r, 0px)); }
/* Enforced floor on interactive/box components (beats a component's 0). */
:where(button,[role="button"],input,select,textarea,.card,.btn,.chip,.badge,dialog,fieldset) {
  border-radius: max(var(--radius-min), var(--_r, var(--radius-md)));
}
.btn, .pill, [data-shape="pill"] { border-radius: var(--radius-full); }
.sharp, [data-sharp] { border-radius: 0 !important; }   /* escape hatch when truly intended */
```
**Tailwind v4 (`@theme`):** set `--radius-none: 0.625rem; --radius-sm/md/lg/xl/2xl/full …` in `@theme`, plus a
`@layer base` `max(0.625rem, …)` floor on interactive elements. **v3:** in `theme.borderRadius` remap
`none:"0.625rem"`, `DEFAULT/md:"0.75rem"`, etc., + the same base-layer floor. (Keep `none:0` + `.sharp`
opt-out only if some elements legitimately need square corners, e.g. full-bleed media/dividers.)

---

## Global reduced-motion baseline (always include)
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration:.01ms !important; animation-iteration-count:1 !important;
    transition-duration:.01ms !important; scroll-behavior:auto !important; }
}
```

## Sources
MDN `contrast-color()`, `light-dark()`, Relative colors · caniuse contrast-color / css-relative-colors ·
WebKit blog + CSS-Tricks (rename from color-contrast; OKLCH `infinity` fallback) · W3C WCAG relative
luminance (0.04045 / coefficients / `(L1+.05)/(L2+.05)`) · una.im + Smashing (self-correcting color).
