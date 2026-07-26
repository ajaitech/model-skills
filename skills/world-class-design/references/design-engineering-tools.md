# Design-engineering tools (2026)

Tools that raise the ceiling. ⌨️ = agent-invocable (CLI/npm/MCP — outputs code or runs in CI); 🖱️ = GUI
(the agent hand-writes the deterministic CSS/SVG the GUI would produce). Verified July 2026.

## Figma → code
- **Figma Dev Mode MCP** ⌨️ — Figma's own MCP; feeds real layout/tokens/variables/component-names to
  agents. **Claude Code integration shipped Feb 2026** (+`download_assets`, local-font render). **Highest
  leverage — reach first when the user has a Figma file + an MCP agent.**
- **Builder.io Visual Copilot** ⌨️+🖱️ — AI Figma→React/Next/Vue/Svelte, maps to *your* components; has a CLI.
- **Anima** 🖱️ — HTML/React/Vue/Tailwind/shadcn/MUI; free React export in Dev Mode.
- **Locofy.ai** 🖱️ — → React/Vue/Angular/Next/RN/Flutter; Figma vars → `:root` custom props.
- **Figma Make** 🖱️ — in-app prompt → interactive prototype with editable code.
- **shadcn/ui Figma kits** 🖱️ — ui.shadcn.com/docs/figma; shadcndesign.com (ships agent skill files),
  shadcnstudio.com, shadcnblocks.com.
- **Tempo (tempo.new)** 🖱️+AI — visual editing on real code, GitHub-ready React.

## Design tokens
- **Style Dictionary** ⌨️ — Amazon's token JSON → CSS/SCSS/JS/iOS/Android/Flutter. **v5.5.0** (ESM;
  keys off DTCG `$type`). `npx style-dictionary build`. Default agent choice.
- **Tokens Studio** 🖱️+⌨️ — Figma plugin, authors tokens, syncs to Git; DTCG default; pairs with Style
  Dictionary via `sd-transforms`.
- **DTCG / W3C format** — first stable `2025.10` (Community Group report, not a Standard). Emit
  `$value`/`$type` for max compatibility.
- **Terrazzo** ⌨️ (ex-Cobalt) — DTCG-native CLI → CSS/Sass/JS/Tailwind + visual Token Lab.
- **Open Props** ⌨️ — ready-made CSS custom props (fluid type/space, colors, easings, shadows, gradients).

## Shadow / gradient / mesh / texture (🖱️ — hand-write the output)
- **Shadows:** Josh Comeau Shadow Palette (layered elevation presets) · shadows.brumm.af (eased smooth shadow).
- **Gradients:** Comeau Gradient Generator (perceptual, fixes gray dead-zone) · cssgradient.io · Hypercolor (Tailwind).
- **Mesh/shader:** MagicPattern · meshgradient.com · Gradienty.
- **SVG bg/blobs/waves/grain:** Haikei (best single source) · Grainient · gggrain (fffuel) · coolbackgrounds.io.
- **Glass:** css.glass · ui.glass/generator.
> Output is deterministic CSS/SVG — hand-write layered `box-shadow`/`linear-gradient` inline; reserve
> Haikei/MagicPattern/Grainient for SVG/raster you genuinely can't synthesize.

## Spacing / grid / type scale
- **Utopia** ⌨️+🖱️ — **the important one**; fluid type+space as `clamp()` from min/max viewport+ratio.
  Programmatic via `utopia-core` (JS/TS) + `tailwind-utopia` plugin.
- **Typescale** (typescale.com) · **Modular Scale** (concept) · **Open Props** (drop-in scales).

## Accessibility auditing (agent order: CI gate → running-app audit → color-pick → human pass)
- **axe-core / @axe-core/cli** ⌨️ — Deque engine. `npx @axe-core/cli <url>` or drive `axe-core` in Playwright.
- **Pa11y / pa11y-ci** ⌨️ — CLI runner; `pa11y-ci` checks a whole sitemap. Cleanest build gate.
- **Lighthouse** ⌨️ — `lighthouse <url> --only-categories=accessibility` (a11y+perf+CWV+SEO).
- **Chrome DevTools MCP** ⌨️ — live Chrome for an agent: `lighthouse_audit`, `performance_start/stop_trace`
  + `performance_analyze_insight` (LCP/CLS/INP), screenshot/console/network. `npx chrome-devtools-mcp@latest`.
  **Best way for a coding agent to actually run a11y+perf on a running app.** (Also available here as the
  `mcp__plugin_chrome-devtools-mcp_chrome-devtools__*` tools + `chrome-devtools-mcp:*` skills.)
- **WebAIM Contrast Checker** 🖱️ (while picking colors) · **WAVE** 🖱️ (in-page "why it fails" overlay).

## Sources
figma.com/blog/introducing-figma-mcp-server · builder.io/figma-to-code · animaapp.com/figma · locofy.ai ·
figma.com/make · ui.shadcn.com/docs/figma · tempo.new · styledictionary.com (v5.5.0) · tokens.studio ·
designtokens.org · terrazzo.app · open-props.style · joshwcomeau.com/shadow-palette + /gradient-generator ·
shadows.brumm.af · magicpattern.design · haikei.app · css.glass · utopia.fyi · typescale.com ·
deque.com/axe + @axe-core/cli · pa11y.org · developer.chrome.com/docs/lighthouse ·
github.com/ChromeDevTools/chrome-devtools-mcp · webaim.org/resources/contrastchecker.
