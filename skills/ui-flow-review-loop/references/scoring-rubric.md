# UI evidence rubric

Mark each applicable category `PASS` only with current source/runtime evidence.
Mark unavailable external evidence `BLOCKED`, never assumed.

| Category | PASS evidence |
|---|---|
| Visual design | Deliberate tokens, typography, hierarchy, spacing, assets, and theme behavior verified in captures. |
| UX and comfort | Primary journey is obvious, minimal, reversible where needed, and provides inline feedback. |
| Functional correctness | Every control and state transition works against the real typed integration path. |
| Validation | Shared pure validators cover malformed, boundary, repeated, and conflicting input. |
| State completeness | Loading, empty, partial, populated, error, retry, and success states are implemented and exercised. |
| Recovery | Failures remain truthful, actionable, resumable, and do not create duplicate effects. |
| Motion | Correct runtime/skill used; motion is purposeful, interruptible, reduced-motion safe, and lifecycle-clean. |
| Consistency | Existing design primitives and behavior owners are reused with no parallel implementation. |
| Responsiveness | Required phone, tablet, desktop, orientation, keyboard/insets, large text, and long-content states pass. |
| Accessibility | Semantics, labels, order, keyboard, focus, contrast, tap targets, and reduced motion pass. |
| Performance | Target profiling shows acceptable frame pacing, paint/layout, memory, network, and asset behavior. |
| Reliability | Offline, timeout, stale response, retry, cancellation, reconnect, race, and restoration paths pass where applicable. |
| Runtime hygiene | Zero console, network, analyzer, lint, type, test, build, or platform warnings/errors. |
| Market readiness | Real data/config, localization, observability, analytics/telemetry boundary, and deployment constraints are complete for scope. |

Closure record per screen:

```text
route:
source revision:
viewports/platforms:
states exercised:
checks:
captures:
blocked:
verdict: PASS | BLOCKED
```
