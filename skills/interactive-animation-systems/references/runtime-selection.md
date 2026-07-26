# Runtime selection

| Requirement | Default | Do not choose when |
|---|---|---|
| Web DOM/SVG UI motion | GSAP | The experience is an authored `.riv` state machine or a rendered game world. |
| Cross-platform interactive vector asset | Rive | Ordinary layout/route animation already belongs to the framework or GSAP. |
| Flutter real-time 2D world | Flame | The need is a normal app micro-interaction or one interactive illustration. |
| Video composition | Installed Remotion skills | The output is an interactive application surface. |

Combine runtimes only across explicit ownership boundaries—for example, GSAP owns
page transitions while an embedded Rive instance owns its internal state machine.
Never let two engines animate the same property or own the same input state.

Before implementation:

1. Identify interaction owner, target platforms, accessibility behavior, lifecycle,
   renderer, asset delivery, and performance budget.
2. Search existing dependencies and primitives; extend the existing owner.
3. Load the one matching skill/reference and exact live API page.
4. Implement lifecycle cleanup, reduced motion, loading/error states, and regression
   coverage with the real runtime.
5. Profile the running target and verify no warnings, leaked resources, duplicate
   requests, off-screen loops, or renderer fallback.
