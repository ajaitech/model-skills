---
name: ui-flow-review-loop
description: Evidence-driven, single-session workflow for building or reviewing a real application UI screen or flow across UX, visual and motion quality, functional correctness, states, accessibility, responsiveness, performance, console, and network behavior. Use for any production UI implementation, visual QA, or screen-by-screen flow review.
---

# UI flow review loop

Use one foreground executor. Do not dispatch subagents or call another model.
Derive the flow graph from actual routing/navigation code, beginning at the real
entry/auth state when applicable. Work one affected node at a time while tracing
its incoming and outgoing states.

## Per-screen loop

1. **Ground:** read the route, screen, reusable primitives, state owner, validators,
   integration calls, data types, assets, tests, and project-knowledge entry.
2. **Enumerate states:** the shared scenario matrix is owned by `elite-launch` — do
   not restate it. Add only the screen-level states it does not cover: populated,
   validation failure, disabled, focus, hover, press, long content, and reduced
   motion where applicable.
3. **Implement sequential lenses:**
   - UX: fewest actions, hierarchy, feedback, recovery, mobile/keyboard comfort.
   - Visual/motion: design-system consistency, typography, spacing, assets, motion,
     focus/hover/active, reduced motion. Route animation through
     `interactive-animation-systems`.
   - Functional: typed state, pure validation/domain rules, real integration,
     idempotency, races, errors, performance, and cleanup.
4. **Run focused checks**, then start the real application.
5. **Verify with browser/platform tooling:** required viewports, text scaling,
   light/dark themes, states, keyboard/insets, semantics, interaction, console,
   network, accessibility, performance, and screenshots.
6. **Evaluate** against
   [references/scoring-rubric.md](references/scoring-rubric.md). Every failed
   criterion becomes a concrete fix, followed by a fresh verification pass.
7. Advance only when all applicable criteria have current evidence and zero errors
   or warnings.

Do not invent a numeric score or claim independent review. The evidence table is
the gate. If a required backend, account, device, or design approval is unavailable,
state the exact unverified criterion instead of substituting a mock.

Source: https://github.com/ajaitech/model-skills/tree/main/skills/ui-flow-review-loop
