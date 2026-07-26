# Animation system routing

| Surface | Load | Boundary |
|---|---|---|
| Web DOM/SVG/component motion | Installed GSAP skill matching the need | GSAP owns DOM/SVG properties, timelines, scroll, drag, FLIP, and framework lifecycle. |
| Rive `.riv` interaction | `interactive-animation-systems` → Rive runtime reference | Rive owns its artboard, state machine, data binding, and renderer lifecycle. |
| Flutter 2D game/simulation world | `interactive-animation-systems` → Flame reference | Flame owns the game loop, world/components, camera, input, collision, effects, and overlays. |
| Native Apple/Android interactive vector | `interactive-animation-systems` → platform Rive reference | Native runtime owns playback, semantics, threading, rendering, and cleanup. |
| Rendered video | Installed Remotion skill matching the task | Remotion owns deterministic frame-driven video composition, not app interaction. |
| Simple platform-native transition | Existing framework/design-system primitive | Do not add an engine for an effect the current stack already owns. |

Shared requirements: one motion-token family, no two engines animating the same
property, reduced-motion behavior, semantic equivalence, interruptible input,
off-screen pause, cached assets, deterministic loading/error states, explicit
dispose/cleanup, target-device profiling, and clean build/console output.

Live Rive and Flame routes:
https://github.com/ajaitech/model-skills/tree/main/skills/interactive-animation-systems
