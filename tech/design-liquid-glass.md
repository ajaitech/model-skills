# Design — Liquid Glass (iOS 27)

Mandatory material system per the org's 22 Strict Rules (Rule 7), authoritative at
https://raw.githubusercontent.com/ajaitech/model-skills/main/AIVIBE_MASTER_AGENT_INDEX.md.
That document overrides any conflict below.

## Applies when
- Any user-facing UI project: `components/`, `screens/`, `lib/widgets/`, or equivalent.
- Mobile (Flutter, React Native, SwiftUI) or web (React/Next) frontend work.
- Any task touching buttons, cards, modals, sheets, toolbars, or empty/loading/error states.

## Authoritative sources
| Need | URL |
|---|---|
| Apple HIG (root) | https://developer.apple.com/design/human-interface-guidelines |
| HIG — Materials | https://developer.apple.com/design/human-interface-guidelines/materials |
| Liquid Glass tech overview | https://developer.apple.com/documentation/TechnologyOverviews/liquid-glass |
| WWDC25 "Meet Liquid Glass" | https://developer.apple.com/videos/play/wwdc2025/219/ |
| Apple Accessibility | https://developer.apple.com/accessibility/ |
| WCAG 2.1 quick reference | https://www.w3.org/WAI/WCAG21/quickref/ |
| Material Design (benchmark) | https://m3.material.io/ |
| Org master index | https://raw.githubusercontent.com/ajaitech/model-skills/main/AIVIBE_MASTER_AGENT_INDEX.md |

## Non-obvious rules
- **Glass is a material, not a color.** A flat semi-transparent fill with no blur is glassmorphism, not Liquid Glass — real backdrop blur (`backdrop-filter: blur() saturate()` or a native blur view) is mandatory.
- **Contrast is computed against the composited result**, not the tint alone. Text passing AA against a "glass gray" swatch can fail against the real photo behind it — derive color from live composited luminance.
- **Depth needs differentiated blur/opacity per layer** — identical blur/opacity across panels reads flat regardless of z-order.
- **The specular edge is what reads as "glass."** A bright ~1px top/leading highlight with a darker trailing edge reads as a lensed layer; uniform borders read as a filtered card.
- **Motion must honor reduced motion** — `prefers-reduced-motion`/"Reduce Motion" must degrade transitions to instant or cross-fade, or it's a regression.
- **"Reduce Transparency" needs a real opaque fallback**, not just disabled blur left on the original low-alpha fill.
- **One pressable component, everywhere** — every clickable surface shares one primitive so hover/press/focus, the 3D lift + tri-color hover inversion, and the 44×44pt hit target stay uniform. Divergent per-screen buttons drift silently.
- **Empty state is a designed state.** No-data screens render a welcome/CTA, never a bare error string. Loading, legitimately-empty, and fetch-failure are three distinct treatments — collapsing them breaks first-run trust.
- **A hard-blocking config read must never leave the screen empty.** An unreadable dependency (e.g. commission/pricing) needs a documented, versioned default so the user can proceed.
- **Permission-gated features never self-trigger** — location/camera/notification prompts fire only on explicit user action with a reason string.

## Production checklist
- [ ] Every glass surface uses real backdrop blur + saturate, not a flat rgba fill
- [ ] Contrast validated against the actual composited background at runtime
- [ ] Depth hierarchy differentiated per layer; specular edge present top/leading
- [ ] Reduced-motion settings respected on every glass transition
- [ ] Reduce Transparency fallback renders solid, same layout and contrast
- [ ] One shared Pressable component used app-wide; 44×44pt minimum hit target
- [ ] Every screen has explicit empty/loading/error designs
- [ ] No feature requests a permission without a preceding user action
- [ ] Hard config dependencies have a documented fallback default

## Never
- Never ship a translucent panel with no backdrop blur and call it Liquid Glass.
- Never hardcode text color per surface instead of deriving it from the composited background.
- Never let a blur/motion transition ignore the OS-level reduced-motion setting.
- Never let two different button/card components diverge in hover, press, or disabled styling.
- Never render a bare error string or blank frame where a first-run empty state belongs.
- Never fire a permission prompt without a preceding explicit user action.
