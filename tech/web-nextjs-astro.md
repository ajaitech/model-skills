# Web: Next.js / Astro

## Applies when
`package.json` lists `next` under dependencies, OR `astro.config.mjs`/`astro.config.ts` exists.

## Authoritative sources
| Source | URL |
|---|---|
| Next.js docs | https://nextjs.org/docs |
| Next.js repo | https://github.com/vercel/next.js |
| Astro docs | https://docs.astro.build |
| Astro repo | https://github.com/withastro/astro |

## Non-obvious rules
- App Router components are Server Components by default. `"use client"` is a boundary directive, not a per-component opt-in flag to sprinkle everywhere — placing it high in the tree drags every child below it into the client bundle. Push it down to the smallest interactive leaf.
- A `"use client"` file can still import and render Server Components passed to it as `children`/props — the boundary is about where the directive is declared, not a hard wall on composition. Don't assume client files can never contain server-rendered content.
- Fetch caching behavior in Next's App Router has changed across versions (default cache-ability of `fetch()` calls, `dynamic` route segment config, `revalidate` semantics). Confirm the installed Next major version's documented default before assuming a request is cached or not — don't carry over a rule from a previous version.
- On-demand cache invalidation goes through `revalidatePath`/`revalidateTag` (Server Actions or Route Handlers only) — time-based `revalidate` alone won't reflect a write that just happened in the same request cycle.
- Route Handlers (`app/api/*/route.ts`) replace the Pages Router's `pages/api/*` — the two conventions are not interchangeable inside the same route tree.
- Middleware runs on the Edge runtime by default — Node-only APIs (`fs`, many npm packages assuming Node globals) fail there with no local warning until deployed.
- Astro ships zero JavaScript to the client by default. A framework component (React/Vue/Svelte) embedded in an `.astro` file renders as static HTML unless given an explicit `client:*` directive (`client:load`, `client:idle`, `client:visible`, etc.) — omitting it is not a bug report, it's the documented default, and it silently kills interactivity.
- Astro "islands" are isolated: state does not automatically share between two islands on the same page, even if they're the same framework — cross-island communication needs an explicit store or event bridge.

## Production checklist
- [ ] Server/Client component boundary reviewed — no client bundle bloat from an unnecessarily high `"use client"`
- [ ] Cache and revalidation strategy set explicitly per route (not left to implicit defaults) for anything showing user-mutated data
- [ ] `generateMetadata`/Astro `<head>` SEO tags present on every public route
- [ ] Every interactive Astro component has the correct `client:*` directive verified in a real browser, not assumed from the code
- [ ] Middleware audited for Node-only API usage if running on Edge
- [ ] Environment variables split correctly between server-only and public/client-exposed

## Never
- Never add `"use client"` to silence a Server Component error without first understanding why the error fired.
- Never assume a framework component inside an `.astro` file is interactive without a `client:*` directive — verify in-browser.
- Never call Node-only APIs from Edge middleware.
- Never fetch or log a secret from a Client Component.
