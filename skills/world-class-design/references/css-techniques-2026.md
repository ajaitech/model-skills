# Cutting-edge CSS techniques (2026) — the award-level frontier

What separates a 10/10 from merely-good. Each entry: verdict, minimal snippet, browser support. **SHIP
NOW** = current Chromium+Firefox+Safari (guard with `@supports`/feature-detect). **PE** = progressive
enhancement only — must degrade to a usable end state, never gate essential content. Verified July 2026.

## View Transitions API
Native crossfade/morph between states. Same-doc (SPA) via `document.startViewTransition()`; cross-doc
(MPA) via `@view-transition`.
- Same-doc: **SHIP NOW** — Baseline Oct 2025 (Chrome 111+, Safari 18+, Firefox 144+).
- Cross-doc: **PE** — Chrome 126+, Safari 18.2+, no Firefox (just navigates normally — free).
```css
.hero { view-transition-name: hero; }            /* unique per rendered element */
@view-transition { navigation: auto; }            /* cross-doc opt-in on both pages */
@media (prefers-reduced-motion: reduce) {         /* VT does NOT auto-respect reduced-motion */
  ::view-transition-group(*), ::view-transition-old(*), ::view-transition-new(*) { animation: none !important; }
}
```
```js
if (document.startViewTransition) document.startViewTransition(() => updateDOM()); else updateDOM();
```

## Scroll-driven animations — **PE ONLY**
Progress tied to scroll (`scroll()`) or visibility (`view()`), off main thread. Chrome 115+, Safari 26+,
Firefox flagged. Always `@supports`-wrap so non-support shows the **end** state (never the invisible "from").
```css
@keyframes reveal { from { opacity:0; transform: translateY(2rem);} to { opacity:1;} }
@supports (animation-timeline: view()) {
  .reveal { animation: reveal linear both; animation-timeline: view(); animation-range: entry 0% cover 40%; }
}
```

## CSS Anchor Positioning — **SHIP NOW + fallback**
Tether tooltips/menus/popovers to an anchor with viewport auto-flip (replaces Floating UI). Chrome 125+,
Safari 26+, Firefox 147+ (~82%). Give a static base `position`, enhance in `@supports`; `@oddbird/css-anchor-positioning` polyfill for legacy.
```css
.trigger { anchor-name: --btn; }
.tooltip { position: fixed; position-anchor: --btn; position-area: block-end center;
           position-try-fallbacks: flip-block, flip-inline; }
```

## `@starting-style` + `transition-behavior: allow-discrete` — **SHIP NOW**
Animate elements entering from `display:none`/top-layer (dialogs, popovers). Baseline Aug 2024 (~91%).
```css
dialog { opacity:1; transition: opacity 300ms, display 300ms allow-discrete; }
@starting-style { dialog[open] { opacity:0; } }   /* enter from */
dialog:not([open]) { opacity:0; }                 /* exit to */
```

## Container queries — **SHIP NOW** (size + `cq*` units)
Components respond to their container, not the viewport. Size queries + `cqi/cqw/cqb` widely available.
(Style queries = PE, Firefox pending.)
```css
.wrap { container-type: inline-size; container-name: card; }
@container card (min-width: 30rem) { .card { grid-template-columns: 1fr 2fr; } }
.card h2 { font-size: clamp(1.25rem, 0.9rem + 2.5cqi, 2rem); }   /* container-relative fluid type */
```

## `text-wrap: balance` / `pretty`
`balance` evens short blocks → **headings/quotes** (SHIP NOW, Baseline). `pretty` kills orphans → **body**
(PE — degrades to normal wrap, no breakage).
```css
h1,h2,blockquote { text-wrap: balance; }
p,li { text-wrap: pretty; }
```

## Popover API — **SHIP NOW** (`auto`/`manual`)
Native top-layer popovers with light-dismiss, focus, `::backdrop`. Baseline 2024 (~91%). Pair with anchor
positioning. (`popover="hint"` = PE → fall back to `auto`.)
```html
<button popovertarget="menu">Open</button>
<div id="menu" popover="auto">…</div>
```

## Subgrid — **SHIP NOW**
Nested grid inherits parent tracks (align card header/body/footer across a row). Baseline Mar 2026 (~97%).
```css
.cards { display: grid; grid-template-columns: repeat(3,1fr); }
.card  { display: grid; grid-row: span 3; grid-template-rows: subgrid; }
```

## Also SHIP NOW
`:has()` (relational selector) · CSS Nesting · `@property` (typed custom props → animatable gradients/
colors: `@property --deg { syntax:"<angle>"; inherits:false; initial-value:0deg; }`) · `field-sizing:
content` (auto-grow inputs). **PE only:** `interpolate-size: allow-keywords` (animate to `height:auto`;
Chromium-only) · CSS Carousel `::scroll-marker`/`::scroll-button` (no Firefox) — but scroll-snap itself ships.

## Verdict table
| Technique | Verdict | Fallback |
|---|---|---|
| View Transitions (same-doc) | SHIP NOW | `if(document.startViewTransition)` → instant |
| View Transitions (cross-doc) | PE | normal nav (automatic) |
| Scroll-driven animations | PE | `@supports`; show end state |
| Anchor positioning | SHIP NOW + fallback | static pos → `@supports`; oddbird polyfill |
| `@starting-style`+`allow-discrete` | SHIP NOW | no-anim show |
| Container size queries + `cq*` | SHIP NOW | — |
| `text-wrap: balance` / Popover auto / Subgrid / `:has()` / Nesting / `@property` / `field-sizing` | SHIP NOW | — |
| `text-wrap: pretty` / style queries / popover hint | PE | normal (automatic) |
| `interpolate-size`, CSS Carousel markers | PE | instant / manual dots |

## Sources
web.dev/blog (same-document-view-transitions-baseline, baseline-entry-animations, popover-api,
at-property-baseline, baseline-digest-mar-2026, web-platform-06-2026) · MDN (View_Transition_API,
@view-transition, Scroll-driven_animations, position-anchor, @starting-style, Container_queries,
field-sizing, ::scroll-marker) · caniuse (css-anchor-positioning, animation-timeline_scroll,
css-text-wrap-balance, css-subgrid, css-nesting) · webkit.org/blog/17101, /16547.
