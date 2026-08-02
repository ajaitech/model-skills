# AivedhA Main Site

## Goal
Marketing + legal site for **AIVEDHA INC** (New York) at `www.aivedha.com`. Literary-magazine ("Editorial Luxe") page whose sole job is routing buyers, crawlers and legal readers to the free beta at `magic.aivedha.com`, a natural-language UI-less ERP.

## Core requirements
- SEO: `index.html` holds canonical, `hreflang` `en`+`x-default`, OG, Twitter Card and 3 JSON-LD blocks (Organization, WebSite, SoftwareApplication). `robots.txt` allows `*`, GPTBot, Google-Extended, ClaudeBot, PerplexityBot; `sitemap.xml` lists the 3 routes.
- A11y/theme: the hero's animated letters are `aria-hidden` with an `sr-only` mirror that must stay in sync (`Hero.tsx:35,67`); icon-only controls carry `aria-label`; `light`/`dark` class on `<html>` from `localStorage['aivedha-theme']` else `prefers-color-scheme` (`ThemeToggle.tsx:10-22`).

## Tech stack
Declared | resolved (`package-lock.json`, `lockfileVersion: 3`).

| Package | Declared | Resolved |
|---|---|---|
| vite | `^6.2.0` | `6.4.2` |
| react, react-dom | `^19.0.0` | `19.2.5` |
| typescript (dev) | `~5.8.2` | `5.8.3` |
| react-router-dom | `^7.1.1` | `7.14.1` |
| tailwindcss, `@tailwindcss/vite` | `^4.1.14` | `4.2.2` |
| express | `^4.21.2` | `4.22.1` |

Node `>=20` (`engines`). The build toolchain (vite, tailwindcss, both plugins) sits in **`dependencies`**, not dev — only `typescript` + `@types/*` are dev, so `npm ci --omit=dev` still builds but `npm run lint` breaks. No database, ORM, Firebase, cloud/payment SDK or auth library, per `package.json` + grep of `src/`.

## Build, run, deploy
| Task | Command |
|---|---|
| Install | `npm ci` |
| Dev / preview | `npx vite` (5173, `host: true`) / `npx vite preview` (4173) |
| Build | `npm run build` → `dist/` |
| Typecheck | `npm run lint` = `tsc --noEmit`; no ESLint |
| Serve prod | `npm run build && npm start` (`node server.mjs`; `PORT`, default 8080) |

**Traps.**
1. `README.md` prescribes `npm run dev` / `npm run preview` — **neither exists**; only `build`, `start`, `lint`. The tree ships no `node_modules/` or `dist/`, so `npm start` before a build 404s on every route.
2. `npm run lint` covers only `["src","vite.config.ts"]` (`tsconfig.json:21`) — **`server.mjs` is checked by nothing**; `@types/express` is installed but unused.
3. `server.mjs:71` uses bare `app.get('*')`; Express 5 rejects it — wildcards must be named, `/*splat`, or `/{*splat}` to also match `/` (expressjs.com/en/guide/migrating-5.html). Do not bump Express without rewriting it.
4. `Masthead.tsx:8-11` and `TypographicDock.tsx:21-22` link bare hashes `#essay`/`#features`, ids that exist only on `Home`; from `/terms` they become `/terms#essay` and do nothing. `Footer.tsx:8-11`'s root-relative `/#essay` works.

## Architecture
Client-rendered SPA; alias `@/*`→`src/*` (`vite.config.ts`). `App.tsx` wraps a `ThemeProvider` and routes `/`→`Home`, `/terms`, `/privacy`, `*`→`Home` — no 404 page, unknown paths render the homepage at 200. `Home.tsx:20-28` stitches 9 sections, Hero→Colophon. Components are PascalCase named exports, but **filename ≠ export in places** (`SectionNumber.tsx`→`SectionHeader`); hooks live in `src/lib/`.

**Outbound refs — two, one fetched.** `src/index.css:1` remotely `@import`s Google Fonts (Fraunces, Manrope, JetBrains Mono). Vite inlines CSS `@import` via postcss-import, which preserves remote URLs, so it survives the build: every page view render-blocks on `fonts.googleapis.com` and pulls fonts from `fonts.gstatic.com` (`preconnect`ed, `index.html:34-35`) — a CSP must allow `style-src`/`font-src` there. `MAGIC_URL` (`src/lib/utils.ts:8`) is only an `<a target="_blank" rel="noopener noreferrer">`, never fetched. `PORT` (`server.mjs:9`) is the only env var read anywhere; no `VITE_*`, no `.env*`.

**Deployment — Google Cloud Run**, per `server.mjs:13` ("Cloud Run proxies"), `.gcloudignore`, and `Privacy.tsx:36` ("Google Cloud Run … us-west1", of the whole AIVEDHA estate). No `Dockerfile`, `cloudbuild.yaml` or CI exists and `.gcloudignore` excludes `dist` + `node_modules`, so a source deploy rebuilds server-side via Google Cloud buildpacks: the Node builder runs `npm run build`, then `scripts.start`, honouring `engines.node` (docs.cloud.google.com/docs/buildpacks/nodejs). The `gcloud run deploy` command, project, region and domain mapping are **unverified — unrecorded in the repo.**

**Design — "Editorial Luxe", not Liquid Glass.** `src/index.css` `@theme`: Fraunces display (variable `opsz`/`wght`/`SOFT`), Manrope, JetBrains Mono; paper `#f5f1e8`, ink `#0c0c0c`, oxblood `#8b1a1a` (`--accent`), brass `#b8935e`; numbered `§` sections, hard offset shadows, grain. The single `backdrop-blur` (`TypographicDock.tsx:43`) is incidental.

**Data.** Nothing is fetched; every model is a TS shape over a hardcoded copy array, declared in the component that renders it — `Entry`, `Act`, `Quote`, `DockItem`, `LegalPageProps`, `Theme`. `STATS`, `NAV`, `COLS`, `SLOGANS`, `CONVERSATION` are untyped. `useDocumentMeta(title, description?)` sets `document.title` + description meta.

## API surface
**No HTTP application API** — no JSON endpoint or form target; grep of `src/` for `fetch(|axios|XMLHttpRequest|WebSocket|EventSource` returns nothing. 8 registrations in `server.mjs`; auth: none.

| `server.mjs` | Behaviour |
|---|---|
| `app.use` :17 | any method/path: host `aivedha.com` → `301` `www.aivedha.com<originalUrl>`, `max-age=3600`; others pass |
| `app.use` :30 | `/assets/*` static, `max-age=365d, immutable` |
| `app.use` :34 | files in `dist/`: `max-age=1h`, `index:false`; `.html`→`no-cache`; sitemap/robots→`max-age=3600`+type; manifest type |
| `app.get` :53 | `GET /healthz` → `200 "ok"` |
| `app.get` :57,61,65 | `/sitemap.xml`, `/robots.txt`, `/manifest.webmanifest` → `res.type()` + `sendFile` |
| `app.get('*')` :71 | everything else → `sendFile dist/index.html`, status **200** |

**Caching gotcha.** `index:false` means HTML at `/`, `/terms`, `/privacy` skips the static middleware; `res.sendFile` (:71) serves it with `maxAge` default `0` → `Cache-Control: public, max-age=0` (expressjs.com/en/4x/api/response/), not the `no-cache` only a literal `/index.html` gets.

## Security, headers & CORS
- Public content only: no auth, accounts, cookies, storage or secrets. No CORS middleware, so no `Access-Control-Allow-*` is emitted and cross-origin reads fail closed — correct today.
- Set: `x-powered-by` off (:12), `trust proxy: true` (:13), tiered `Cache-Control`. Absent — **GAP for a public origin**: CSP, HSTS, `X-Content-Type-Options`, `X-Frame-Options`/`frame-ancestors`, `Referrer-Policy`, `Permissions-Policy`. So the site can be framed anywhere (the `magic.aivedha.com` CTA is clickjackable), responses are MIME-sniffable, a first plain-HTTP hit is not TLS-pinned, and script/style origins are unconstrained.
- The redirect reads client-controllable `X-Forwarded-Host` under unconditional `trust proxy` (:18). **Not** an open redirect — the target is the literal `https://www.aivedha.com` (:21) — but a spoofed header forces a `301` on any request.
- ERP OAuth/data logic lives in `magic.aivedha.com`; `Privacy.tsx` describes the *combined* AIVEDHA surface (Cognito/Google/GitHub sign-in, Zoho tokens, Gemini processor), none of it enforced here.

## Known gaps & risks
- `README.md` is stale three ways: its tree omits `HowItWorks.tsx`, `Testimonials.tsx`, `lib/useDocumentMeta.ts`; its Development block lists dead scripts; it says "§ 01…§ 06" when live markers are § 01, 02, 02½, 03, 04, 04½, 05.
- `Testimonials.tsx:14-43` hardcodes 4 named individuals with companies, roles and cities as customer quotes via `dangerouslySetInnerHTML` over a static template literal (not XSS-exploitable); line 90 adds "— submitted under real names and consent —". **Neither the attribution nor that consent claim is verifiable from the repo — reputational and advertising-claims risk. Confirm written consent, or de-identify, before further publication.**
- `*`→`Home` returns a 200 homepage for any unknown URL, including `/api/*` (which `robots.txt` disallows though it does not exist); no soft-404 signal.
- No tests or test runner; `noUnusedLocals`/`noUnusedParameters` are `false`, so dead code slips through. No CI either — release is manual and undocumented.
- `sitemap.xml` `lastmod` and both legal pages' `effective` are hardcoded to 2026-04-14; nothing regenerates them.
