# The 10/10 checklist — what an award-winning screen has that a good one lacks

A "good" screen is clean and functional. A 10/10 screen is **decided** — every state, edge, and pixel is
intentional. Gate every screen against all ten before calling it done. (Complements the
`ui-flow-review-loop` adjudicator rubric.)

1. **Visual hierarchy is unmistakable.** One clear focal point per view; size/weight/color/space encode
   importance (not decoration); 3–4 type sizes max on a modular scale; `text-wrap: balance` on headings,
   `pretty` on body; whitespace as an active element. *Good lacks:* competing focal points, uniform
   emphasis, cramped rhythm.
2. **Motion is deliberate and interruptible.** One motion family (shared easing/duration tokens); GPU-only
   props at 60fps; enter fast (≤150ms) / exit relaxed; springs for physical things; shared-element
   (`layoutId`) for continuity; **nothing animates above-the-fold on first paint**; reduced-motion honored.
   *Good lacks:* random per-component curves, load-in staging, jank on `height`/`top`.
3. **Empty / loading / error / success states are crafted.** Empty states teach + have a primary CTA;
   loading uses **skeletons that match final layout** (no spinner soup, no layout shift); errors are
   specific + recoverable + inline next to the trigger; optimistic UI with rollback; a real success moment.
   *Good lacks:* blank zero-states, generic "Something went wrong."
4. **Keyboard + a11y are real.** Full keyboard operability; visible `:focus-visible` rings (via `--ring`);
   logical focus order + trap/return in dialogs; semantic HTML + ARIA only where needed; tap targets ≥44px;
   **axe/pa11y clean**; every text/bg pair passes WCAG AA in both themes. *Good lacks:* mouse-only
   affordances, invisible focus, muted text that fails contrast.
5. **Core Web Vitals pass (field p75).** **LCP ≤2.5s · INP ≤200ms · CLS ≤0.1.** Explicit image dimensions
   (no CLS), responsive `srcset`/AVIF-WebP, lazy-load below fold, preload the LCP asset, `font-display:swap`,
   code-split. Verify with Lighthouse / DevTools MCP trace. *Good lacks:* hero images that shift layout,
   unbudgeted JS, untested vitals.
6. **Copywriting is product design.** Specific human microcopy; buttons name the outcome ("Create project",
   not "Submit"); empty/error/tooltip copy guides; consistent voice; no lorem ipsum shipped. *Good lacks:*
   generic labels, jargon, filler.
7. **Dark mode is first-class, not inverted.** Dark-first tokens via `light-dark()`; elevation via lighter
   surfaces (not just shadow); reduced chroma on large dark fills; imagery/shadows re-tuned for dark; both
   themes contrast-verified. *Good lacks:* a `filter: invert` afterthought, glowing pure-white text.
8. **Responsive + fluid + container-aware.** Fluid type/space via `clamp()`/`cqi` (Utopia), not a few
   breakpoints; components respond to their **container** (size queries); subgrid for cross-card alignment;
   verified 320px → ultrawide; **on mobile compact aggressively — multiple cards per row, never one
   full-width card per row** (`responsive-mobile-compact`). *Good lacks:* desktop-only polish, breakpoint
   jumps, mobile one-card stacking.
9. **Consistency by construction (system, not screens).** One token layer (color/space/type/radius/shadow/
   motion); one primitive library, one icon set, zero duplicate implementations; the three mandatory rules
   baked in (min-radius, 3D triple-invert hover, auto-contrast text). *Good lacks:* off-scale values, two
   button systems, mixed icon stroke weights.
10. **A moment of delight (earned, not sprinkled).** One or two signature touches — a satisfying transition,
    an origin-aware hover, a scroll reveal, a considered illustration/mesh background, a delightful success
    animation — intentional, never impeding the task, degrading gracefully under reduced-motion. *Good
    lacks:* zero personality, or delight-spam that slows everything down.

**One-line gate:** hierarchy obvious · motion deliberate + reduced-motion-safe · empty/loading/error/success
all crafted · keyboard + AA verified both themes · CWV pass (LCP 2.5 / INP 200 / CLS 0.1) · copy specific ·
dark-mode native · fluid + container-aware + mobile-compact · one system zero duplicates · one earned moment
of delight.
