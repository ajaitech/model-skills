# AivedhA Main Site

## Goal
Corporate marketing and legal site for **AIVEDHA INC** (New York, USA) at `www.aivedha.com`. A single-page, literary-magazine-styled ("Editorial Luxe") landing page whose sole conversion goal is sending visitors to the free beta of the actual product, `magic.aivedha.com` — a natural-language, UI-less ERP. Audience: prospective enterprise buyers (CFOs, founders, operators), search/AI crawlers, and Terms/Privacy readers. Source: `README.md`, `index.html`.

## Core requirements
- Builds to a static `dist/` bundle deployable to any static host (`README.md`).
- `/terms`, `/privacy` serve as real routes via SPA fallback, not hash links (`server.mjs`, `App.tsx`).
- SEO: canonical URL, hreflang, Open Graph, Twitter Card, 3 JSON-LD blocks (`Organization`, `WebSite`, `SoftwareApplication`) in `index.html`.
- SEO: `robots.txt` allows `GPTBot`, `Google-Extended`, `ClaudeBot`, `PerplexityBot` + `*`; disallows `/api/` (non-existent).
- SEO: `sitemap.xml` lists `/`, `/terms`, `/privacy` with `lastmod`.
- Perf: `/assets/*` → `Cache-Control: max-age=365d, immutable`; HTML → `no-cache` (`server.mjs`) — must survive hosting swap.
- Perf: Google Fonts `preconnect`ed; `magic.aivedha.com` `dns-prefetch`ed.
- A11y: hero letter animation duplicated in `sr-only` span (`Hero.tsx`); icon-only controls carry `aria-label`.
- A11y/UX: `color-scheme: light dark` + `theme-color` metas; theme persists via `localStorage['aivedha-theme']`, falls back to `prefers-color-scheme`.
- Site must never call a backend — only outbound reference is `magic.aivedha.com`.
- Node `>=20` required (`package.json` `engines`).

## Tech stack
| Layer | Technology | Version (exact) | Source of truth |
|---|---|---|---|
| Build tool | Vite | `^6.2.0` | `package.json` |
| UI framework | React / react-dom | `^19.0.0` | `package.json` |
| Language | TypeScript | `~5.8.2` | `package.json` (dev) |
| Routing | react-router-dom | `^7.1.1` | `package.json` |
| Styling | Tailwind CSS + Vite plugin | `^4.1.14` | `package.json`, `vite.config.ts` |
| Animation | motion | `^12.23.24` | `package.json` |
| Icons | lucide-react | `^0.546.0` | `package.json` |
| Class utils | clsx / tailwind-merge | `^2.1.1` / `^3.5.0` | `package.json` |
| Prod server | Express | `^4.21.2` | `package.json`, `server.mjs` |
| Type defs | @types/{express,node,react,react-dom} | `^4.17.21`/`^22.14.0`/`^19.0.0`/`^19.0.0` | `package.json` |
| Runtime | Node.js | `>=20` | `package.json` `engines` |
| Lockfile | npm | `lockfileVersion: 3` | `package-lock.json` |

## Architecture
Pure client-rendered SPA, no backend API. `vite.config.ts` builds `src/` into static `dist/`; alias `@`→`./src`. `src/main.tsx` mounts `<App/>` in `<StrictMode><BrowserRouter>`. `src/App.tsx`: `ThemeProvider`→`GrainOverlay`→`Masthead`→routed `<main>`→`Footer`→fixed `TypographicDock`. Routes: `/`→`Home`, `/terms`→`Terms`, `/privacy`→`Privacy`, `*`→`Home`. `src/pages/Home.tsx` stitches 9 sections: `Hero, Marquee, Essay, Specimen, HowItWorks, FreeIssue, FeaturesIndex, Testimonials, Colophon`.

`server.mjs` is a static file server, not an app backend: 301 redirects `aivedha.com`→`https://www.aivedha.com` (preserving path/query), serves `dist/` with tiered `Cache-Control`, exposes `/healthz`, re-serves `sitemap.xml`/`robots.txt`/`manifest.webmanifest` with explicit content-types, falls back to `dist/index.html` for SPA routes.

**Deployment target**: Google Cloud Run — inferred from `server.mjs` (`trust proxy: true`, "Cloud Run proxies" comment) + `.gcloudignore` (no Dockerfile, likely buildpack deploy running `npm start`); corroborated by `Privacy.tsx` clause 4: "operated on Google Cloud Run ... (us-west1)." No `cloudbuild.yaml`/CI in-repo — **unverified**.

Only outbound reference anywhere: `https://magic.aivedha.com` (`MAGIC_URL` in `src/lib/utils.ts`), opened `target="_blank"` — never fetched or proxied.

## Naming conventions
| Kind | Convention | Example |
|---|---|---|
| Component files | PascalCase, filename = export | `FeaturesIndex.tsx` exports `FeaturesIndex` |
| Page files | PascalCase under `src/pages/` | `Terms.tsx` exports `Terms` |
| Hook files | camelCase, `use`-prefixed | `useDocumentMeta.ts` |
| Lib files | camelCase | `utils.ts` exports `cn()` |
| Constants | SCREAMING_SNAKE_CASE | `export const MAGIC_URL = '...'` |
| CSS classes | kebab-case, editorial-themed | `.display-hero`, `.editorial-rule`, `.paper-grain` |
| Theme tokens | kebab-case custom props under `@theme` | `--color-oxblood`, `--font-display` |
| localStorage keys | kebab-case, app-prefixed | `aivedha-theme` |
| Routes | lowercase, no trailing slash | `/`, `/terms`, `/privacy` |
| Import alias | `@/*` → `src/*` | `import { cn } from '@/lib/utils'` |
| Env vars | none app-specific; only `PORT` | `server.mjs` |

## Data types & models
Static content site — **no database, no runtime fetching**. "Models" are TypeScript interfaces for hardcoded in-source content arrays.

| Entity | Fields (name : type) | Store | Defined in |
|---|---|---|---|
| `Entry` (feature row) | `roman,title,italic,body,page : string` | in-memory array | `FeaturesIndex.tsx` |
| `Act` (how-it-works step) | `numeral,kicker,title,italic,body,tag,duration : string` | in-memory array | `HowItWorks.tsx` |
| `Quote` (testimonial) | `body,who,role,city,rule : string` | in-memory array | `Testimonials.tsx` |
| `Stat` (colophon counter) | `value:string, numeric:number\|null, label:string` | in-memory array | `Colophon.tsx` |
| `DockItem` (bottom nav) | `roman,label:string, to?,href?,hash?:string, accent?:bool` | in-memory array | `TypographicDock.tsx` |
| `LegalPageProps` | `kicker,title,subtitle,effective,metaDescription:string, children:ReactNode` | props | `LegalPage.tsx` |
| `Theme` | `'light' \| 'dark'` | context + `localStorage['aivedha-theme']` | `ThemeToggle.tsx` |

## API surface
**No HTTP application API.** `server.mjs` only serves static files + a health probe — no JSON endpoint, no form target, no `fetch`/`axios`/`XMLHttpRequest` in `src/` (repo-wide grep verified).

| Operation | Path | Request | Response | Auth | Defined in |
|---|---|---|---|---|---|
| Health check | `GET /healthz` | none | `200 "ok"` text | none | `server.mjs` |
| Canonical redirect | any, host `aivedha.com` | none | `301` → `https://www.aivedha.com<path+query>` | none | `server.mjs` |
| Static assets | `GET /assets/*` | none | file, `max-age=365d, immutable` | none | `server.mjs` |
| Static root files | `GET /*` (file match) | none | file, `max-age=1h` (`no-cache` `.html`) | none | `server.mjs` |
| Sitemap/robots/manifest | `/sitemap.xml`, `/robots.txt`, `/manifest.webmanifest` | none | file, explicit content-type | none | `server.mjs` |
| SPA fallback | `GET *` (no file match) | none | `dist/index.html` | none | `server.mjs` |

Outbound third-party call: none programmatic. Only external reference is a plain `<a target="_blank">` to `https://magic.aivedha.com` — a separate product.

## CORS & headers
No CORS middleware or `Access-Control-*` headers — **none configured — GAP** (moot today, relevant if `magic.aivedha.com` ever fetches this origin).
Set in `server.mjs`: `X-Powered-By` disabled; `trust proxy: true`; tiered `Cache-Control`. No CSP, HSTS, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy` — **not configured — GAP** for a public origin.

## Security boundary
- Entirely public content. No auth, no user accounts, no server-side data storage in this repo.
- No secrets read anywhere except the non-sensitive `PORT` env var (`server.mjs`); no `.env*` files present.
- `README.md`: "The site never talks to a backend" — ERP OAuth/data/auth logic lives in the separate `magic.aivedha.com` app, outside this repo.
- Trust assumption: `trust proxy: true` + `X-Forwarded-Host` assumes the process sits only behind Cloud Run's proxy; no infra-as-code in-repo to confirm.
- Privacy Policy documents data practices for the *combined* AIVEDHA surface (this site + `magic.aivedha.com`); that logic is outside this repo — here it's documentation only.

## Known gaps & risks
- `README.md` component tree is stale — omits `HowItWorks.tsx`, `Testimonials.tsx`, `useDocumentMeta.ts`, all wired into `Home.tsx`.
- No security headers (CSP/HSTS/X-Frame-Options/X-Content-Type-Options/Referrer-Policy) — verified absent from `server.mjs`.
- No CORS policy configured.
- `Testimonials.tsx` hardcodes 4 named individuals/companies/cities as quotes via `dangerouslySetInnerHTML`; static (not XSS-exploitable), sourcing/consent unverifiable.
- No automated tests anywhere (no `*.test.*`/`*.spec.*`, no test runner).
- No lint config; `npm run lint` = `tsc --noEmit` only; `noUnusedLocals`/`noUnusedParameters` are `false`, so dead code isn't caught.
- No CI workflow (`.github/` absent), no `cloudbuild.yaml`/`Dockerfile` despite `.gcloudignore` implying `gcloud` deploy.
- `robots.txt` disallows `/api/`, a path that doesn't exist — vestigial.
