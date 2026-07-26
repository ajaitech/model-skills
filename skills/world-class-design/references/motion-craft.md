# Motion & micro-interaction craft (2026)

Motion reads *premium* not from more, but because it obeys physics, responds instantly, and is
interruptible. Libraries live in `component-libraries.md`; this is the craft. Verified July 2026.

## Runtime routing

Load the implementation authority before writing motion code:

| Need | Authority |
|---|---|
| Web DOM/SVG motion | Installed `gsap-core`; add only the matching timeline, ScrollTrigger, plugin, framework, utility, or performance GSAP skill. |
| Rive state machine/data-bound vector | [`interactive-animation-systems`](../../interactive-animation-systems/SKILL.md) and its exact platform Rive reference. |
| Flutter 2D game/simulation world | [`interactive-animation-systems`](../../interactive-animation-systems/SKILL.md) and its Flame reference. |
| Deterministic rendered video | Installed Remotion skill matching the task. |
| Simple native transition | The project's existing framework/design-system primitive. |

One runtime owns each animated property. Never duplicate GSAP guidance here, never
make two engines drive the same state, and never add an engine when the current
stack already owns the behavior.

## What the authorities say
- **Emil Kowalski (animations.dev):** `ease-out` for user-initiated enter/exit; keep under **~300ms**
  (500ms enter is too slow); **never animate frequent keyboard actions** (delightful once = frustrating
  100×/day); **spring for anything physical**; animate **only `transform`+`opacity`** (composite-only);
  **interruptible**; avoid `ease-in` (sluggish start); prefer **custom curves** (built-ins too weak).
- **Rauno Freiberg (interfaces.rauno.me):** interaction ≤**200ms**; scale **proportionally** (dialogs
  from ~0.8, buttons to ~0.96, never to 0); looping anims **pause off-screen**; theme-switch triggers no
  transitions; hover gated by `@media (hover:hover)`; input **font-size ≥16px** (no iOS zoom); **optimistic
  UI**; feedback inline next to its trigger (checkmark, not toast); dropdowns open on **`mousedown`**.
- **Josh Comeau:** default **~250ms**; **asymmetric hover — enter 125–150ms, exit ~450ms**; avoid
  animating `height`/`width`/`left`/`background-color`; dropdown-flicker fix via exit `transition-delay`.
- **Val Head:** animation improves feedback, aids orientation, directs attention, shows causality,
  expresses brand. **Choreography** = one shared easing/duration family.
- **Apple (WWDC18 "Fluid Interfaces"):** springs are **interruptible + velocity-aware** — start from the
  current on-screen value, inherit velocity, grabbable/reversible mid-interaction.
- **Material 3:** spatial springs (position — may overshoot) vs effects springs (color/opacity — must NOT
  overshoot). Token curves below are from M3 live docs.

## Drop-in tokens
```css
:root {
  /* Material 3 (verified) */
  --ease-standard: cubic-bezier(.2,0,0,1);
  --ease-emphasized-decel: cubic-bezier(.05,.7,.1,1);   /* ENTER */
  --ease-emphasized-accel: cubic-bezier(.3,0,.8,.15);   /* EXIT  */
  /* Premium custom (stronger decel) */
  --ease-out:    cubic-bezier(.16,1,.3,1);     /* enter */
  --ease-in:     cubic-bezier(.4,0,1,1);       /* exit only when ending off-screen */
  --ease-in-out: cubic-bezier(.65,0,.35,1);    /* reposition/morph on-screen */
  --ease-spring: cubic-bezier(.34,1.56,.64,1); /* CSS-only overshoot */
  /* Durations — scale with travel distance + element size */
  --dur-instant:50ms; --dur-micro:100ms; --dur-fast:150ms;      /* hover/press ≤200ms */
  --dur-enter:250ms; --dur-exit:200ms; --dur-hover-out:450ms;   /* component enter/exit */
  --dur-layout:450ms; --dur-full:500ms;                          /* page/sheet */
}
```

## Springs, stagger, layout (Motion)
- **Spring presets:** `snappy {stiffness:500,damping:32,mass:1}` (toggles/small) · `smooth {240,28,1}`
  (default, no bounce) · `bouncy {320,18,1}` (playful) · native `{ visualDuration:0.35, bounce:0.2 }`.
  Spatial → spring (may overshoot); color/opacity → tween or non-overshoot spring.
- **Stagger:** children **20–60ms** apart, parent-first, whole cascade <~400ms (cap long lists):
  `transition:{ delayChildren:0.1, staggerChildren:0.04 }`.
- **Layout:** Motion `layout` (FLIP), `layoutId` (shared-element "magic move" — the premium tell),
  `<AnimatePresence mode="popLayout">` for exit while siblings reflow (children need `forwardRef`).
- **Hover/press:** instant feedback **<100ms**; press faster than release; scale-down on press **~0.97**.
- **Scroll-telling:** reveal-on-enter (once, fade + rise 8–16px) is safe; long pinned narratives are costly.

## The "premium tells"
Interruptible (continue from current value, never snap-restart) · origin-aware (`transform-origin`/
`originX/Y` at the pointer — menus scale out of their trigger) · **nothing animates on above-the-fold
first paint** (hero is *there*, not staged) · spring for physical/grabbable · **GPU-only props, hold
60fps** (touching `height`/`top`/`width` is a downgrade tell) · **one motion family** (shared tokens —
random per-component curves are the #1 amateur tell).

## Reduced-motion (never optional — swap movement for opacity, keep instant, never broken)
```css
@media (prefers-reduced-motion: reduce) {
  *,*::before,*::after { animation-duration:.01ms!important; animation-iteration-count:1!important;
    transition-duration:.01ms!important; scroll-behavior:auto!important; }
}
```
Motion: gate variants behind `useReducedMotion()` / `<MotionConfig reducedMotion="user">`.

## Sources
emilkowal.ski/ui/great-animations · animations.dev/learn/animation-theory/the-easing-blueprint ·
interfaces.rauno.me (github.com/raunofreiberg/interfaces) · joshwcomeau.com/animation/css-transitions +
/a-friendly-introduction-to-spring-physics · rosenfeldmedia (Designing Interface Animation) +
alistapart.com/article/designing-interface-animation · developer.apple.com/videos/play/wwdc2018/803 ·
m3.material.io/styles/motion/easing-and-duration/tokens-specs · motion.dev/docs (react-transitions,
react-layout-animations, react-animate-presence).
