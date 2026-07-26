---
name: live-official-docs
description: Use when writing, upgrading, debugging, reviewing, or configuring Laravel, Flutter, Dart, TypeScript, Python, Firebase, Google Cloud or gcloud, AWS, Razorpay, PayPal, GitHub, Figma, or Canva code, APIs, CLIs, SDKs, integrations, or syntax.
---

# Live official documentation

Use the repository's installed versions as the compatibility boundary and current first-party documentation as the knowledge source. Do not rely on remembered release facts.

## Retrieval contract

1. Inspect manifests, lockfiles, tool versions, and existing code first.
2. Read only the matching reference file below.
3. Open the exact canonical API, language reference, command reference, or user-guide page first; normally use one to three URLs. Follow links only inside the listed official domains.
4. Use release notes, changelogs, and migration pages only as secondary evidence for an upgrade, deprecation, regression, or installed-to-target difference. They are never the skill, the primary implementation source, or sufficient without the exact API/reference page.
5. Use newer syntax only when compatible with the installed version, or after an explicitly requested upgrade and migration review.
6. Cite the exact official pages used. If live access fails, state what remains unverified.

Do not preload every reference, copy vendor documentation into skills, or use blogs, search snippets, generated summaries, or third-party tutorials as authority.

## Routes

| Task | Read |
|---|---|
| Laravel | [references/laravel.md](references/laravel.md) |
| Flutter | [references/flutter.md](references/flutter.md) |
| Dart | [references/dart.md](references/dart.md) |
| TypeScript | [references/typescript.md](references/typescript.md) |
| Python | [references/python.md](references/python.md) |
| Firebase | [references/firebase.md](references/firebase.md) |
| Google Cloud / gcloud | [references/google-cloud.md](references/google-cloud.md) |
| AWS | [references/aws.md](references/aws.md) |
| Razorpay | [references/razorpay.md](references/razorpay.md) |
| PayPal | [references/paypal.md](references/paypal.md) |
| GitHub | [references/github.md](references/github.md) |
| Figma | [references/figma.md](references/figma.md) |
| Canva | [references/canva.md](references/canva.md) |

Rive, Flame, and GSAP are intentionally not duplicated here. Route animation work
through `interactive-animation-systems`, which delegates web motion to the installed
GSAP skills and loads only the matching official Rive or Flame page.

Source: https://github.com/ajaitech/model-skills/tree/main/skills/live-official-docs
