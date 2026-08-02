# AivedhA Main Site

## Goal
Corporate marketing + legal site for **AIVEDHA INC** (New York, USA) at `www.aivedha.com`. Literary-magazine ("Editorial Luxe") landing page whose sole conversion goal is sending visitors to the free beta of the product, `magic.aivedha.com` — a natural-language, UI-less ERP. Audience: enterprise buyers, AI/search crawlers, legal-page readers.

## Core requirements
- Static `dist/` deployable to any static host; `/terms` and `/privacy` are real routes via SPA fallback, not hashes.
- SEO in `index.html`: canonical, `hreflang` `en`+`x-default`, Open Graph, Twitter Card, 3 JSON-LD blocks. `public/robots.txt` allows `*`, `GPTBot`, `Google-Extended`, `ClaudeBot`, `PerplexityBot`; `public/sitemap.xml` lists the 3 routes.
- Perf: `/assets/*` → `max-age=365d, immutable`; `.html` → `no-cache` — must survive a hosting swap. Node `>=20`.
- A11y: hero letter animation mirrored in an `sr-only` span (`Hero.tsx`); icon-only controls carry `aria-label`.
- Theme: `color-scheme: light dark` + paired `theme-color` metas; persisted in `localStorage['aivedha-theme']`, else `prefers-color-scheme`.
- Never call a backend — the only outbound reference is `magic.aivedha.com`.

## Tech stack
Declared | resolved in `package-lock.json` (`lockfileVersion: 3`).

| Layer | Technology | Declared | Resolved |
|---|---|---|---|
| Build | Vite | `^6.2.0` | `6.4.2` |
| UI | react / react-dom | `^19.0.0` | `19.2.5` |
| Language | TypeScript (dev dep) | `~5.8.2` | `5.8.3` |
| Routing | react-router-dom | `^7.1.1` | `7.14.1` |
| Styling | tailwindcss + `@tailwindcss/vite` | `^4.1.14` | `4.2.2` |
| Server | Express | `^4.21.2` | `4.22.1` |
| Other | motion `^12.23.24`, lucide-react `^0.546.0`, clsx `^2.1.1`, tailwind-merge `^3.5.0`, @vitejs/plugin-react `^5.0.4` | — | — |

No database, ORM, Firebase, AWS/GCP SDK, payment SDK or auth library — verified in `package.json` and by grep of `src/`.

## Build, run, deploy
| Task | Command |
|---|---|
| Install | `npm ci` |
| Dev / preview | `npx vite` (5173, `host: true`) / `npx vite preview` (4173) |
| Build | `npm run build` → `dist/` |
| Typecheck | `npm run lint` = `tsc --noEmit`; no ESLint config |
| Serve prod | `npm run build && npm start` (`node server.mjs`, `PORT`, default 8080) |

**Traps.** (1) `README.md` says run `npm run dev` / `npm run preview` — **neither exists**; only `build`, `start`, `lint` are defined. (2) `npm start` serves `dist/`; without a prior build every route `sendFile`s a missing `dist/index.html`. (3) `server.mjs` uses the bare wildcard `app.get('*')`, which Express 5 rejects — it requires the named `'/{*splat}'` (expressjs.com/en/guide/migrating-5.html); do not bump to Express 5 without rewriting that route.

## Architecture
Client-rendered SPA, no backend API. `vite.config.ts`: `react()` + `tailwindcss()`, import alias `@/*`→`src/*`. `src/main.tsx` mounts `<App/>` in `<StrictMode><BrowserRouter>`. `src/App.tsx`: `ThemeProvider` → `GrainOverlay` → `Masthead` → `ScrollToTop` → routed `<main id="main">` → `Footer` → fixed `TypographicDock`. Routes `/`→`Home`, `/terms`→`Terms`, `/privacy`→`Privacy`, `*`→`Home` (no 404 page; unknown paths render the homepage at 200). `Home.tsx` stitches 9 sections: `Hero, Marquee, Essay, Specimen, HowItWorks, FreeIssue, FeaturesIndex, Testimonials, Colophon`.

`server.mjs` is a static file server, not an app backend — see API surface. Only outbound reference: `MAGIC_URL = 'https://magic.aivedha.com'` (`src/lib/utils.ts`), rendered as `<a target="_blank">` — never fetched or proxied.

**Deployment — Google Cloud Run**, grounded in the `server.mjs` "Cloud Run proxies" comment, `.gcloudignore`, and `Privacy.tsx` clause 4 ("operated on Google Cloud Run ... us-west1"). No `Dockerfile`, `cloudbuild.yaml` or CI in-repo, so this is a source buildpack deploy: the GCP Node.js buildpack runs `npm run build` when a `build` script exists, then `scripts.start`, honouring `engines.node` (docs.cloud.google.com/docs/buildpacks/nodejs). The exact `gcloud run deploy` invocation is **unverified — absent from the repo**.

## Design system
"Editorial Luxe", **not** Liquid Glass. `src/index.css` `@theme`: `--font-display` Fraunces (variable `opsz`/`wght`/`SOFT`), `--font-sans` Manrope, `--font-mono` JetBrains Mono; paper `#f5f1e8`, ink `#0c0c0c`, `--color-oxblood` `#8b1a1a` (= `--accent`), `--color-oxblood-deep` `#5f0f0f`. Numbered `§` sections, asymmetric broadsheet layout, hard offset shadows, paper grain. The repo's single `backdrop-blur` (`TypographicDock.tsx`) is incidental — do not apply Liquid Glass rules here.

## Naming conventions
| Kind | Convention | Example |
|---|---|---|
| Components / pages | PascalCase, filename = export | `FeaturesIndex.tsx` |
| Hooks | camelCase, `use`-prefixed | `useDocumentMeta.ts` |
| Constants | SCREAMING_SNAKE_CASE | `MAGIC_URL`, `STATS` |
| CSS classes / theme tokens | kebab-case, editorial-themed | `.paper-grain`, `--color-oxblood` |
| localStorage keys | kebab-case, app-prefixed | `aivedha-theme` |

## Data types & models
"Models" are TypeScript shapes over hardcoded in-source arrays.

| Entity | Fields | Defined in |
|---|---|---|
| `Entry` (feature row) | `roman,title,italic,body,page : string` | `FeaturesIndex.tsx` |
| `Act` (how-it-works step) | `numeral,kicker,title,italic,body,tag,duration : string` | `HowItWorks.tsx` |
| `Quote` (testimonial) | `body,who,role,city,rule : string` | `Testimonials.tsx` |
| `DockItem` (bottom nav) | `roman,label:string, to?,href?,hash?:string, accent?:bool` | `TypographicDock.tsx` |
| `LegalPageProps` | `kicker,title,subtitle,effective,metaDescription:string, children` | `LegalPage.tsx` |
| `Theme` | `'light' \| 'dark'`; ctx `{theme, toggle}` | `ThemeToggle.tsx` |
| `STATS` (colophon counters) | `value:string, numeric:number\|null, label:string` — untyped literal, **no named interface** | `Colophon.tsx` |

`useDocumentMeta(title, description?)` sets `document.title` and upserts `<meta name="description">`; used by `Home`, `LegalPage`.

## API surface
**No HTTP application API.** No JSON endpoint, no form target, no `fetch`/`axios`/`XMLHttpRequest` in `src/` (grep verified). All 6 operations live in `server.mjs`; auth: none.

| Operation | Path | Response |
|---|---|---|
| Health | `GET /healthz` | `200 "ok"` |
| Canonical redirect | any path, host `aivedha.com` | `301` → `www.aivedha.com<path+query>`, `max-age=3600` |
| Static assets | `GET /assets/*` | `max-age=365d, immutable` |
| Static root | `GET /*` (file match) | `max-age=1h`; `.html` → `no-cache` |
| Sitemap/robots/manifest | those 3 | explicit content-type |
| SPA fallback | `GET *` (no file) | `dist/index.html` |

## Headers & CORS
Set: `X-Powered-By` disabled, `trust proxy: true`, tiered `Cache-Control`. Absent — **GAP** for a public origin: CSP, HSTS, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`. No CORS middleware — GAP, moot until another origin fetches this one.

## Security boundary
- Entirely public content. No auth, accounts, or server-side storage. No secrets; only the non-sensitive `PORT` env var is read, and no `.env*` file exists.
- ERP OAuth/data logic lives in the separate `magic.aivedha.com` app. `Privacy.tsx` documents the *combined* AIVEDHA surface; enforcement lives elsewhere.
- `trust proxy: true` + `X-Forwarded-Host` assumes the process sits only behind Cloud Run's proxy; nothing in-repo confirms that.

## Known gaps & risks
- `README.md` is stale twice: its component tree omits `HowItWorks.tsx`, `Testimonials.tsx`, `useDocumentMeta.ts`, and its Development section lists scripts that do not exist.
- `Testimonials.tsx` hardcodes 4 named individuals with named companies, roles and cities as customer quotes, via `dangerouslySetInnerHTML` over a static template literal (not XSS-exploitable). **Sourcing and consent are unverifiable from the repo — reputational and advertising-claims risk. Confirm attribution before further publication.**
- `*`→`Home` returns a 200 homepage for any unknown URL; no soft-404 signal for crawlers.
- No tests or test runner; `noUnusedLocals`/`noUnusedParameters` are `false`, so dead code is not caught.
- No CI workflow, `Dockerfile` or `cloudbuild.yaml` despite `.gcloudignore` implying a `gcloud` source deploy — release is manual and undocumented.
- `robots.txt` disallows `/api/`, a non-existent path. `sitemap.xml` `lastmod` is hardcoded `2026-04-14`; nothing regenerates it.
