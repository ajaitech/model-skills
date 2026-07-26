---
name: design-assets
description: >-
  Use when a UI needs any visual asset — photos, stock images, illustrations, icons, animations,
  motion graphics, video, fonts, or logos. The sourcing pipeline so agents FETCH world-class assets
  fast instead of hand-struggling or leaving gray boxes: free public stock (with exact fetch
  recipes) plus the internal home-grown S3 stock. Also the logo-recolor-before-use rule and the
  division of labor where asset legality is handled separately (not by the designer). Use for
  image/photo/illustration/icon/animation/Lottie/Rive/motion/video/font/logo needs together with
  world-class-design and ui-flow-review-loop; use interactive-animation-systems for runtime code.
---

# Design Assets — fetch world-class, never struggle

## Overview

Agents must **stop wasting time and tokens** trying to hand-draw illustrations, invent logos, or
leave placeholder gray boxes. Every visual asset is **fetched** from a curated source and used —
provided it clears the world-class benchmark. There are two sources, chosen **only** by which gives
the best visual treat to the application:

1. **Internal S3 stock (home-grown)** — an immediate, pre-configured library on your own S3. Treat it
   exactly like any external stock site; it just happens to be internal. Structure and fetch commands
   below.
2. **Free public sources** — the best free places for each asset type, each with a concrete fetch
   recipe (API/CDN/npm/download), in [`references/free-sources.md`](references/free-sources.md).

**The decision rule is visual merit only.** Choose home-grown vs public purely on which asset makes
the application look best — never because one is "internal" or "external." If neither the internal
stock nor the recommended sources satisfy the benchmark, the agent may do its own additional research
to find a better asset.

---

## Division of labor — legality is NOT the designer's job

During design, the agent's focus is **100% visual excellence and go-to-market speed**. Do **not**
spend tokens, steps, or context assessing usage rights, licensing, or legality of an asset while
designing — that is handled by a **separate process the user runs personally after hosting/going
live**. Select assets by visual fit; keep momentum toward launch. (In practice the sources in
`free-sources.md` are already free/CC0/free-tier, so this rarely arises — but the rule stands: rights
clearance is out of scope for the design pass.)

---

## Internal S3 stock

Path and per-subfolder fetch commands are in
[`references/free-sources.md`](references/free-sources.md) (the "Internal S3 stock" section). The
stock is organized into these subfolders:

| Subfolder | Holds |
|---|---|
| `icons/` | icon sets / individual icons |
| `images/` | photographs, stock images, backgrounds |
| `themes/` | theme presets, palettes, style kits |
| `graphics/` | graphic elements, shapes, patterns, textures |
| `videos/` | motion backgrounds, clips |
| `illustrations/` | spot & scene illustrations (prefer recolorable SVG) |
| `fonts/` | display & text typefaces |
| `logos/` | app-name-specific ready logos (see the logo rule below) |

**How it's used:** browse the relevant subfolder; if an asset meets the world-class benchmark, **copy
it into the project directory** and use it as a normal project asset. The S3 stock is a reference
shelf, not a runtime dependency — nothing links to S3 at runtime; the chosen asset is pulled in.

### Logos — recolor BEFORE applying (strong rule)
The `logos/` folder holds ready, app-name-specific logo files so agents **do not waste time creating
logos**. But a stock logo is **never applied as-is**. Before using it you **must**:
1. **Recolor the logo** to match the application's theme/palette.
2. **Fix its background** — remove it (transparent) or recolor it — so it sits perfectly on the app's
   surfaces.
Only after recoloring + background treatment does the logo go into the app. A logo whose colors clash
with the theme is a defect. (Recolor techniques for SVG/PNG logos are in `references/free-sources.md`.)

---

## Rule

For any visual need: check the internal S3 stock and the matching source in `free-sources.md`, fetch
the best-fitting asset, recolor/treat it to the theme (mandatory for logos), copy it into the project,
and move on. No gray boxes, no hand-drawing from zero, no rights-analysis during design.

## Self-check
Every visual need met by a fetched asset (no gray boxes / no from-scratch illustration) · picked by
visual merit only (home vs public irrelevant) · logo recolored + background-treated to the theme
before use · assets copied into the project (not linked to S3 at runtime) · zero tokens spent on
rights/legality during the design pass.

Source: https://github.com/ajaitech/model-skills/tree/main/skills/design-assets
