---
name: interactive-animation-systems
description: Route production animation work to the installed GSAP skills, current Rive runtime documentation, or Flame Engine. Use for web motion, scroll animation, Flutter interactive graphics, Rive state machines and data binding, native Apple or Android Rive integration, renderer selection, and Flutter 2D game or simulation scenes.
---

# Interactive animation systems

Choose the smallest runtime that matches the real interaction model. Inspect the
project manifest, lockfile, target platforms, current renderer, asset pipeline, and
existing animation primitives before adding a dependency.

## Route

| Need | Authority |
|---|---|
| DOM/SVG motion, timelines, scroll, drag, FLIP, React/Vue/Svelte | Installed `gsap-core`, `gsap-timeline`, `gsap-scrolltrigger`, `gsap-plugins`, `gsap-react`, `gsap-frameworks`, `gsap-utils`, and `gsap-performance` skills |
| Authored interactive vector graphics, state machines, data binding, cross-platform `.riv` assets | [references/rive.md](references/rive.md) |
| Flutter 2D game loops, components, collision, particles, camera, world, or overlays | [references/flame.md](references/flame.md) |
| Choosing or combining runtimes | [references/runtime-selection.md](references/runtime-selection.md) |

Do not vendor or restate the GSAP skills. Load only the matching skill. For Rive
or Flame, open one to three exact official URLs from the matching reference; use
the Rive `llms.txt` index only to discover a page not already routed.

## Production contract

- Reuse the project's current motion tokens and primitives; one owner per behavior.
- Respect reduced motion, semantics, focus, touch, lifecycle, pause/resume, and
  resource cleanup.
- Keep animation state connected to real domain/UI state; never use fake data,
  placeholder controls, or a visual-only success path.
- Select renderers from current official guidance and benchmark the actual target.
- Pause off-screen work, avoid duplicate asset loads, dispose native/WASM resources,
  and verify frame pacing, memory, console, and platform build output.
- Use test fixtures only inside tests. Production must load real owned assets through
  the existing asset/config path.

Source: https://github.com/ajaitech/model-skills/tree/main/skills/interactive-animation-systems
