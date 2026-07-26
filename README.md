# Model Skills

Private source of truth for AiVibe-owned skills used by Claude Code, Codex,
Gemini CLI, Antigravity, and VS Code.

- Full skill content exists once under `skills/`.
- [`loaders/model-skills/SKILL.md`](loaders/model-skills/SKILL.md) is the single
  compact runtime router installed into each model.
- Vendor skills such as GSAP and Remotion are not copied here; the router points
  to their installed authoritative skill names.
- Official vendor documentation is referenced by exact live URL and crawl
  context. Changelogs are compatibility-only.

Repository: https://github.com/ajaitech/model-skills
