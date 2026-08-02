# Web: TypeScript + React + Vite

## Applies when
`package.json` lists both `react` and `vite` under dependencies/devDependencies.

## Authoritative sources
| Source | URL |
|---|---|
| TypeScript docs | https://www.typescriptlang.org/docs |
| Vite docs | https://vitejs.dev |
| Vite repo | https://github.com/vitejs/vite |
| React docs | https://react.dev |
| React repo | https://github.com/facebook/react |

## Non-obvious rules
- Only env vars prefixed `VITE_` are exposed to client code via `import.meta.env`. The common production mistake runs the opposite direction from what it sounds like: a secret accidentally prefixed `VITE_` (e.g. `VITE_STRIPE_SECRET_KEY`) IS bundled into shipped client JS and is world-readable. Prefix only values safe for a browser to see.
- `import.meta.env.*` values are statically replaced at build time, not read at runtime. Deploying the same build artifact to multiple environments with different env values does not work — env-dependent config needs a runtime fetch or per-environment build.
- Vite does not type-check by default; `vite build` succeeds even with TypeScript errors (esbuild strips types without checking them). CI must run `tsc --noEmit` (or a checker plugin) separately, or type errors ship to production silently.
- `"strict": true` in `tsconfig.json` is not automatically what a fresh Vite template gives you at every preset — verify it's actually on; it gates `noImplicitAny`, `strictNullChecks`, and related flags as a bundle, not individually.
- React 18+ batches state updates automatically outside event handlers too (timeouts, promises, native handlers) — code written assuming React 17's narrower batching can silently coalesce updates a developer expected to be synchronous.
- `React.StrictMode` double-invokes component bodies and certain effects in development only, to surface impure logic — this is not a bug and does not happen in production builds.
- Vite pre-bundles dependencies with esbuild on first dev-server start (cached in `node_modules/.vite`); a stale cache after a dependency version bump causes confusing runtime errors — clear it before assuming a real bug.
- The Vite dev server's built-in proxy (`server.proxy`) exists only in dev; production deployments need the hosting layer (reverse proxy/CDN rule) to handle the same API routing, or CORS breaks post-deploy.

## Production checklist
- [ ] `tsc --noEmit` passes in CI, independent of `vite build` succeeding
- [ ] No secret/private key is prefixed `VITE_`
- [ ] `.env.production` (or hosting-provider env config) reviewed for values that differ from dev
- [ ] Source maps stripped from the public build, or uploaded privately to an error tracker only
- [ ] Route-level `import()` code splitting in place for non-trivial bundle size
- [ ] Bundle size checked against a budget (`vite build --report` / bundle analyzer)
- [ ] CSP and other security headers configured at the hosting layer

## Never
- Never prefix a secret, private key, or server-only credential with `VITE_`.
- Never treat a successful `vite build` as proof of type safety — it doesn't type-check.
- Never rely on dev-only globals (HMR APIs, `import.meta.hot`) executing unguarded in code that ships to production.
- Never commit `.env` files containing real secrets, even ones without the `VITE_` prefix.
