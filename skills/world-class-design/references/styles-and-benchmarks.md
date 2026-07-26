# 2026 Styles & Benchmarks — aim above the default

Consult these BEFORE building so the first draft targets the current visual ceiling, not a templated
look. Sourced from 2026 trend reviews (Figma, Gezar, StudioMeyer).

## Top styles (how to achieve + reference exemplar)
| Style | How to achieve (one line) | Exemplar |
|---|---|---|
| **Bento grids** | Asymmetric grid of rounded cards, varied col/row-span; biggest card = primary message. | Apple, Vercel, Linear |
| **Depth / 3D & soft shadows** | Layered low-opacity multi-shadows + subtle perspective/rotate on hover; no harsh drops. | Stripe, Family |
| **Aurora / mesh gradients** | Large blurred color blobs (`filter: blur(120px)`) slowly animated behind a dark section. | Stripe, Linear, Vercel |
| **Glass (evolved)** | `backdrop-filter: blur()` + low-opacity bg + hairline border; restrained, on nav/modals over dark. | Apple, macOS/iOS |
| **Kinetic / large display type** | Oversized variable-font headings; animate weight/position on scroll. **Use sparingly** (mostly demo-ware). | Awwwards SOTD, Godly |
| **Tasteful motion** | 150-300ms transitions, spring/ease-out, `AnimatePresence`; motion signals state, never decorates. | Linear, Vercel, Family |
| **Dark-first** | Tokens dark by default (OKLCH), light as override; dark makes glass/aurora/depth read cleanest. | Linear, Vercel |

> Reality check (StudioMeyer): bento, restrained glass, aurora, and dark-first shipped to production;
> kinetic typography mostly stays on Awwwards demos — use it sparingly.

## Benchmark sources (consult before building)
| Source | Best for |
|---|---|
| **Mobbin** (mobbin.com) | Real shipped app UI by flow — production UX patterns, not concepts. |
| **Refero** (refero.design) | Real SaaS/web-product screenshots by page type & component. |
| **Awwwards** (awwwards.com) | The visual ceiling — award-judged animation/immersive/technical showcases. |
| **Godly** (godly.website) | Tightly curated motion/aesthetic web design — evaluate motion fast. |
| **Land-book** (land-book.com) | Curated landing pages; filter to one section (pricing, features) when stuck. |
| **SaaS Landing Page** (saaslandingpage.com) | Real SaaS landing pages — section/structure benchmarking. |
| **Dribbble** (dribbble.com) | Palette, type pairing, illustration ideas (concepts, not production). |
| **Screenlane** (screenlane.com) | Web/mobile UI pattern screenshots (onboarding, settings, empty states). |

**Workflow:** Mobbin + Refero (UX/real-product) → Awwwards + Godly (visual bar) → Land-book / SaaS
Landing Page (section structure) → Dribbble / Screenlane (palette/pattern detail). Aim to **beat** the
references, not match them.

## More sources by need (deep bench)
`($)` = subscription. Reach for the specific one, don't scroll full pages.

- **Mobile app UI:** Mobbin ($) · **ScreensDesign** ($, real iOS screens/onboarding/**paywalls**;
  successor to UI Sources & Scrnshts) · **Page Flows** ($, recorded end-to-end flow videos) · Nicelydone
  ($) · Collect UI (free) · Pttrns ($).
- **Dashboards / SaaS:** **SaaSFrame** ($, marketing vs product split; empty-state section) · SaaS
  Interface ($) · **SaaSUI** (saasui.design — pattern *reasoning*; tracks Attio/Hex/Linear/Vercel) · Refero.
- **Motion / interaction:** Awwwards `/websites/motion/` · Godly · **The FWA** (thefwa.com — WebGL/scroll-
  story ceiling) · Muzli (muz.li feed).
- **Award bodies / juried:** **CSS Design Awards** (daily WOTD scored /10) · The FWA · Webby · Lovie ·
  **SiteInspire** (minimalist taste) · Httpster (brutalist/typographic) · Minimal Gallery.
- **Curation / moodboard:** **Savee** (savee.com — designer default; Figma plugin) · Cosmos (cosmos.so) ·
  Are.na · **Cofolios** (portfolios of designers who landed Apple/Meta/Figma) · Layers (layers.to).
- **Component/pattern level:** UI Patterns (rationale) · Design Vault · **The Component Gallery** (single-
  component variants) · Empty States (emptystat.es) · Pricing Pages · Unsection (by section: hero/pricing).

Redirects: uisources.com & scrnshts.club → screensdesign.com; savee.it → savee.com.

## Sources
figma.com/resource-library/web-design-trends · gezar.dk/en/blog/web-design-trends-2026 ·
studiomeyer.io/en/blog/webdesign-trends-2026-reality-check · socialscript.in/blog/design-inspiration-sites-for-2026 ·
pageflows.com · screensdesign.com · saasframe.io · saasui.design · cssdesignawards.com · thefwa.com ·
siteinspire.com · savee.com · cofolios.com · ui-patterns.com · component.gallery · unsection.com.
