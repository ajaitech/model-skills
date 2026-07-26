# TypeScript official sources

Match `package.json`, the lockfile, local `tsc --version`, and `tsconfig.json`. Do not apply a newer compiler's defaults to an older project.

| Need | Official URL | Crawl context |
|---|---|---|
| Language and type-system syntax | https://www.typescriptlang.org/docs/ | Open the exact Handbook section and verify examples against the installed compiler. |
| Compiler configuration | https://www.typescriptlang.org/tsconfig/ | Verify each option's availability, default, interactions, and deprecation status. |
| Compatibility only: release and breaking changes | https://www.typescriptlang.org/docs/handbook/release-notes/overview.html | After the Handbook/reference page, check installed-to-target changes affecting code or compiler options. |
| Compatibility only: tagged releases | https://github.com/microsoft/TypeScript/releases | Use only to confirm official artifacts or a release detail absent from the exact docs page. |

Official domains: `typescriptlang.org` and `github.com/microsoft/TypeScript`.
