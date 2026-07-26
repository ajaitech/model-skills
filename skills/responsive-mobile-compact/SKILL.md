---
name: responsive-mobile-compact
description: MANDATORY design rule for ANY responsive UI, mobile layout, card grid, dashboard, marketing section, or frontend component (build OR review). Load BEFORE writing/adjusting Tailwind/CSS layout or breakpoints. On mobile, compact aggressively and keep MULTIPLE cards per row — never collapse a card grid to a single full-width card per row (the user calls one-card-per-row / word-by-word stacking "pigs vomit", unprofessional, forbidden). Scale down font sizes, padding, gaps and icons on small screens so more content is visible with less scrolling.
---

# Responsive = compact-on-mobile, not single-column stacking

The user's standing rule (zero tolerance). Default frameworks stack everything to one
column on mobile — that is REJECTED here. Mobile must stay dense and professional.

## Hard rules

1. **Never one card per row on mobile.** Card/feature/tier/logo grids show **at least 2
   per row** on phones. Small stat/metric/number cards show **3 per row**. Only genuinely
   full-width things (a single hero, a form, a long paragraph) may be one per row.
   - Bad:  `grid gap-6 md:grid-cols-2`   (→ 1 column on mobile)
   - Good: `grid grid-cols-2 gap-3 md:gap-5`  (2 on mobile, 2 on desktop)
   - Good: `grid grid-cols-3 gap-2 md:grid-cols-3 md:gap-5`  (compact stat cards)

2. **Mobile-first compaction — base classes are the small/tight values; `md:` scales UP.**
   Every size that matters gets a smaller mobile value:
   - Type: `text-2xl md:text-5xl`, `text-sm md:text-lg`, body `text-[11px] md:text-sm`.
   - Padding: `p-3.5 md:p-6`, section `py-11 md:py-24`.
   - Gaps: `gap-2 md:gap-5`, block spacing `mt-10 md:mt-14`.
   - Icons: `h-9 w-9 md:h-12 md:w-12`, glyphs `h-5 w-5 md:h-6 md:w-6`.
   - Radius: `rounded-2xl md:rounded-3xl`, never below the Rule 2 minimum.

3. **Cramped-but-readable cards on mobile:** stack icon-on-top with `flex flex-col …
   md:flex-row` so text keeps full width in a narrow 2-col cell; tighten line-height
   (`leading-[1.45]`), trim copy so two columns don't overflow.

4. **Minimize scroll & maximize visibility.** The whole component should show more items
   per screen on mobile than desktop-scaled type would. Shorten long paragraphs for the
   mobile cell; hide non-essential decorative glyphs on mobile (`hidden md:block`).

5. **The 22 rules still apply at every breakpoint.** This skill only adds mobile density;
   it never relaxes them. Compaction must not be used to justify breaking any rule.

## Verify before "done"

Always screenshot the real page at **≤ 390px width** (Playwright/chrome-devtools) and
confirm: 2+ cards per row, no giant single-column stack, text legible, no horizontal
overflow, fewer scroll-lengths than the naive stack. One-per-row on mobile = FAIL, redo.

## Apply everywhere

This is global: every new component, every review, every project — proactively apply it,
don't wait to be told. If existing code stacks to one column on mobile, fix it.
