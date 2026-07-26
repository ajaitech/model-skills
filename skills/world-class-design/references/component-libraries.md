# Component Libraries — best-of-breed, composed without duplication (2026)

Verified July 2026. **The best component may come from a different library than the rest** — nearly
all ship as copy-in source (you own the code), so mix freely; the only discipline is **no duplicate
implementation of the same behavior**. Items needing a version re-check are marked [verify].

## Best-of-breed by need

### Foundation / distribution
- **shadcn/ui** — the base layer + distribution CLI (copies component source into your repo). CLI is
  **v4 (Mar 2026)**, defaults to **Base UI** primitives (Radix still supported). Registry + namespace
  model lets you pull from many registries into one project. Tailwind + CSS-variable tokens
  (`--background/--foreground/--primary/--radius`, OKLCH under Tailwind v4).
  `npx shadcn@latest init` → `npx shadcn@latest add button dialog …`
- **HeroUI (ex-NextUI)** — batteries-included, beautiful-by-default set. **v3 (2026)** is a rewrite on
  **React Aria + Tailwind v4**, compound API, no `<Provider>`. `npm i @heroui/react @heroui/styles`.

### Accessible primitives (behavior layer — pick EXACTLY ONE)
- **Base UI (MUI)** — the modern, actively-maintained primitive layer; **v1.0 stable (Dec 2025)**;
  shadcn's new default. `npm i @base-ui-components/react` [verify: migrating to `@base-ui/react`].
- **Radix Primitives** — the incumbent, powers most existing shadcn code; mature, safe. Per-primitive
  install e.g. `npm i @radix-ui/react-dialog`. (Acquired by WorkOS; velocity slowed on combobox/multiselect.)
- **React Aria (Adobe)** — accessibility-rigorous hooks + `react-aria-components` (40+ patterns, best
  i18n/ARIA). `npm i react-aria-components`. Choose when a11y is a hard requirement.

> One primitive source only — never ship two dialog/popover/dropdown implementations.

### Animated / marketing (all Motion-based → one engine, no conflict)
- **Aceternity UI** — high-impact hero/landing effects (3D cards, aurora, spotlight, beams).
  `npx shadcn@latest add @aceternity/<name>` (add registry to `components.json`).
- **Magic UI** — 50+ animated marketing micro-interactions. `npx shadcn@latest add @magicui/<name>`.
- **Motion Primitives** — tasteful, composable motion (text effects, morphing dialogs) without the
  template look. `npx shadcn@latest add "https://motion-primitives.com/c/<name>.json"`.
- **Cult UI** — curated headless + animated shadcn-compatible pieces. `@cult-ui/<name>`.
- **Kokonut UI** — production-ready animated blocks. `@kokonutui/<name>`.

### Broad sets
- **COSS UI (ex-Origin UI)** — 400+ Tailwind copy-in components, rebased on Base UI; Cal.com's design
  system. coss.com/ui [verify registry URL].
- **Park UI** — framework-agnostic (React/Vue/Solid/Svelte) on **Ark UI** + **Panda CSS** (build-time,
  typed tokens). `npx @park-ui/cli init` → `add button`.

### Dashboards & data-viz (also see `dataviz` skill)
- **Tremor** — dashboard components (KPI cards, charts) on Tailwind + Recharts. Copy-paste (tremor.so)
  or `npm i @tremor/react`.
- **Recharts** — the practical default chart engine (v3, 2026). shadcn charts + Tremor both wrap it →
  standardize on it. `npm i recharts`.
- **visx (Airbnb)** — low-level D3-in-React for bespoke viz only. `npm i @visx/visx`.
- **Embla** — the carousel primitive (shadcn Carousel uses it). `npm i embla-carousel-react`.

## Compose across libraries WITHOUT duplication — the 3-layer recipe
1. **Primitive/behavior layer — exactly one** (Base UI *or* Radix *or* React Aria). Every dialog,
   popover, dropdown, tooltip, combobox resolves to that one source.
2. **Token/styling layer — one.** Tailwind + one CSS-variable set + one `cn()` util (`@/lib/utils`),
   so mixed-origin components look like one system.
3. **Composed/visual layer — many registries, deduped.** shadcn is the spine; layer Aceternity / Magic
   UI / Cult / Kokonut / Motion Primitives into the same `components/ui` tree.

```bash
npx shadcn@latest init
npx shadcn@latest add button card dialog dropdown-menu tooltip carousel
# add @aceternity/@magicui/@cult-ui registries to components.json, then:
npx shadcn@latest add @aceternity/bento-grid @magicui/animated-beam @cult-ui/texture-card
npx shadcn@latest add --dry-run <registry/comp>   # preview files/deps → catch a smuggled-in duplicate primitive
```

**Dedup rules (the "avoid overlaps" list):**
- Dialog / Popover / Dropdown / Tooltip / Combobox → primitive layer ONLY. If a marketing registry
  bundles its own, keep yours and delete theirs (you own the files — a real delete).
- Charts → one engine. shadcn charts + Tremor = Recharts (fine). visx only for the specific custom chart.
- Carousel → Embla (shadcn Carousel). Don't also add Swiper/react-slick.
- Animation → one engine (all the above use Motion). Don't additionally wire React Spring for them.
- Icons → one set (Lucide is the shadcn/Park-UI default). Two icon libs = inconsistent stroke weights.

## Animation / motion — which to reach for
| Library | Best for | Install | Minimal usage |
|---|---|---|---|
| **Motion** (ex-Framer Motion) | 90% of React UI animation; layout/gesture, `AnimatePresence`. Default. | `npm i motion` | `import { motion } from "motion/react"` |
| **GSAP** | Scripted timelines, scroll-choreography, SVG/canvas. **100% free** (all plugins) since 2024. | `npm i gsap @gsap/react` | `useGSAP(() => gsap.to(".box",{x:100}))` |
| **AutoAnimate** | One-line list/reorder/enter-exit transitions. | `npm i @formkit/auto-animate` | `const [parent] = useAutoAnimate()` |
| **Lenis** | Smooth scroll + scroll-driven effects (pairs w/ GSAP). | `npm i lenis` | `<ReactLenis root />` (retired: `@studio-freight/react-lenis`) |
| **React Spring** | Physics/spring interpolation. | `npm i @react-spring/web` | `useSpring({opacity:1, from:{opacity:0}})` |

**Reduced-motion is mandatory** (see design-rules-css.md global baseline). Motion: `<MotionConfig
reducedMotion="user">` / `useReducedMotion()`. GSAP: `gsap.matchMedia()` on `(prefers-reduced-motion:
no-preference)`.

## Start-at-world-class method (before the first component)
1. **Tokens first** — OKLCH semantic color (dark-first, light as override), one base `--radius`
   (0.625rem) with sm/md/lg/xl derived, elevation/shadow tokens.
2. **8pt spacing** — base 8px, half 4px; snap all padding/margin/gap to 4/8/12/16/24/32/48/64 (Tailwind
   default scale already encodes this — no arbitrary `p-[13px]`).
3. **Type scale** — one modular scale (~1.25 Major Third): 12/14/16/20/25/31/39/48 → semantic tokens;
   line-height ~1.5 body, ~1.1 display; one display + one text family.
4. **Motion tokens** — 150ms micro / 200-300ms standard / 400-500ms large; ease-out enter, ease-in exit;
   motion conveys state, never decorates; reduced-motion wired day one.
5. **Pick one primitive layer + one icon set (Lucide).**
6. **Pre-build checklist:** benchmarked 3-5 refs (Mobbin/Refero/Awwwards) · palette+tokens committed ·
   one primitive layer · shadcn init + registries added · layout intent + focal point decided · a11y
   baseline (reduced-motion, focus-visible via `--ring`, contrast) · dedup rule acknowledged.

If all are true before the first `<div>`, the first render is system-driven, on-scale, dark-first,
intentionally animated, and accessible — world-class **by construction**, not by rescue.

## Sources
ui.shadcn.com (cli/registry/changelog) · base-ui.com · radix-ui.com · react-spectrum.adobe.com/react-aria ·
heroui.com · ui.aceternity.com · github.com/magicuidesign/magicui · motion-primitives.com · cult-ui.com ·
kokonutui.com · coss.com/ui · park-ui.com · tremor.so · recharts.org · airbnb.io/visx · embla-carousel.com ·
motion.dev · gsap.com · auto-animate.formkit.com · github.com/darkroomengineering/lenis · react-spring.dev.
