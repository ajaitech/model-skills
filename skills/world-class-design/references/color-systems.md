# Color systems & palette engineering (2026)

**Stop hand-picking hex.** A palette is a *scale*: hold hue ~constant, march **lightness** in even
perceptual steps, peak **chroma** in mid-tones. Generate in **OKLCH**, give each step a *semantic job*,
gate every text/bg pair on a contrast checker, build **dark-first** and override light via `light-dark()`.
Extends the auto-contrast rule (D3) in `design-rules-css.md`. Verified July 2026.

## Why OKLCH beats HSL/RGB
Perceptually uniform L (equal steps *look* equal); no hue shift when C or L changes; wider gamut (P3/
Rec.2020); native CSS. This is what makes a generated ramp look even instead of muddy.

## Tools (generate → verify → preview)
| Tool | Use |
|---|---|
| **Radix Colors** (`@radix-ui/colors`) | The canonical **12-step semantic scale** (light+dark+alpha). Copy the *step jobs* (below) as your token contract. |
| **Leonardo** (`@adobe/leonardo-contrast-colors`) | Generate ramps **from target contrast ratios** (ratio is the input). Use when "must be 4.5:1 on that surface" is a hard requirement. |
| **Reasonable Colors** (reasonablecolors.com) | WCAG-AA-by-construction; shade-gap ⇒ contrast (Δ2≥3:1, Δ3≥4.5:1, Δ4≥7:1). Ship fast without a checker. |
| **oklch.com** (Evil Martians) | OKLCH picker with live sRGB/P3/Rec.2020 gamut. |
| **Huetone** (huetone.ardov.me) | Hues×tones matrix showing **WCAG + APCA** per cell. The free "is every step accessible?" workbench. |
| **Tailwind v4** | Whole default palette in OKLCH (22 hues × 50→950). Reuse its slate/blue L·C curves as a template. |
| **Generators** | Accessible Palette, InclusiveColors (exports Tailwind/CSS/Figma), uicolors.app (50→950 from one hex), Realtime Colors (live on a real layout). |

## Radix 12-step semantic jobs (memorize — this is the token contract)
1 app bg · 2 subtle bg · 3 component bg (rest) · 4 hover · 5 pressed/selected · 6 subtle border/
separator · 7 interactive border · 8 strong border / focus ring · **9 solid** (highest chroma —
buttons/badges) · 10 solid hover · 11 muted text · 12 high-contrast text. Steps 11/12 hit APCA Lc 60/90
on step-2 bg. **Bright hues (Sky/Mint/Lime/Yellow/Amber) use DARK text on 9–10.**

## Recipe — perceptually-uniform, accessible, dark-first
1. **Hue once per family**, hold ±few°; vary L as the primary axis; give C a curve peaking mid-tones,
   easing to 0 at extremes. (Neutrals get a tiny chroma at the brand hue → intentional grays.)
2. **Generate the scale** (Tailwind 50→950 or Radix 12-step). Tailwind L·C template: L% descends
   ~98→13 smoothly; C rises to a peak at 500–600 then falls; H within a few degrees.
3. **Semantic tokens** via `light-dark(LIGHT, DARK)` so components bind to **roles**, not raw shades.
4. **State colors** own a hue each (success≈150, warning≈75, danger≈27, info≈230), same L·C curve; the
   solid step must clear ≥4.5:1 with its on-color — bright hues need **dark on-text**.
5. **Verify** every text/bg + every `--on-*` pair in **both** modes (Huetone/WebAIM); APCA as refinement.

```css
:root {
  color-scheme: light dark;
  --n-50:oklch(98.4% .003 257); --n-200:oklch(92.9% .013 257); --n-500:oklch(55.4% .046 257);
  --n-700:oklch(37.2% .044 257); --n-900:oklch(20.8% .042 266); --n-950:oklch(12.9% .042 265); --white:oklch(100% 0 0);
  --b-400:oklch(70.7% .165 255); --b-500:oklch(62.3% .214 260); --b-600:oklch(54.6% .245 263); --b-700:oklch(48.8% .243 264);
  --warning-solid:oklch(80% .16 75); --on-warning:var(--n-950);   /* DARK text on bright hue */
  --danger-solid:oklch(55% .21 27);  --on-danger:var(--white);
  --bg:light-dark(var(--n-50),var(--n-950)); --surface:light-dark(var(--white),var(--n-900));
  --border:light-dark(var(--n-300,#cbd5e1),var(--n-700,#334155));
  --primary:light-dark(var(--b-600),var(--b-600)); --primary-hover:light-dark(var(--b-700),var(--b-500)); --on-primary:var(--white);
  --text:light-dark(var(--n-900),var(--n-50)); --text-muted:light-dark(var(--n-500),var(--n-400));
  --ring:light-dark(var(--b-600),var(--b-400));
}
@supports not (color: light-dark(#000,#fff)) {
  :root { --bg:var(--n-950); --text:var(--n-50); }
  @media (prefers-color-scheme: light) { :root { --bg:var(--n-50); --text:var(--n-900); } }
}
```
Tailwind v4: raw ramps in `@theme { --color-brand-500: oklch(…) }`; keep the semantic `--bg/--text/
--primary` layer as plain custom props so components bind to roles.

## Contrast standard (the stance)
Ship **WCAG 2.x AA as the pass/fail gate** — body ≥4.5:1, large (≥24px, or ≥18.66px bold) ≥3:1, UI/graphic
≥3:1. Use **APCA Lc ≥60 body / ≥90 strong** as a refinement (superior but not normative in 2026 — WCAG 3.0
is a Working Draft, APCA currently "exploratory"). Satisfy both to hedge.

> The per-step lightness numbers here are engineering starting points from the Tailwind/Radix curves —
> tune against Huetone/WebAIM, don't treat as exact published constants.

## Sources
radix-ui.com/colors (+ /docs/palette-composition/understanding-the-scale) · leonardocolor.io +
github.com/adobe/leonardo · reasonablecolors.com · oklch.com + evilmartians.com/chronicles/oklch-in-css ·
huetone.ardov.me · tailwindcss.com/docs/colors · accessiblepalette.com · inclusivecolors.com · uicolors.app ·
realtimecolors.com · w3.org/TR/wcag-3.0 + adrianroselli.com/2026/04/wcag3-contrast-as-of-april-2026 ·
MDN color_value/light-dark.
