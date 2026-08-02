# Aicippy.io

## Goal
Production backend-as-a-service — managed Postgres, auth, storage, realtime, edge functions — rebranded off a Supabase-derived estate into a fully AiCippy-owned surface, billed via PayPal. Sites `aicippy.io`, `docs.aicippy.io`, `studio.aicippy.io`.

Acceptance gates (`ACCEPTANCE.md`): zero residual `supabase`, `supa-`, `stripe` strings in owned code/docs/routes; PayPal the only payment provider; `auth.aivibe.cloud` (Cognito) and `api.aivibe.cloud` (GraphQL/AppSync) reachable with authz server-side; build, lint, typecheck, tests, e2e, visual regression and Lighthouse CI all green.

## Tech stack
From the `aicippy/pnpm-workspace.yaml` catalog and `aicippy/package.json` unless noted.
- pnpm workspaces + Turborepo: pnpm `11.8.0`, turbo `2.9.14`, Node `>=24 <25` (`.nvmrc` = `24`).
- Next.js `16.2.10` across apps `studio`, `www`, `docs`, `design-system`, `ui-library`; React/react-dom `^19.2.7`.
- TypeScript `~5.9.3` monorepo / `~5.8.3` in `aicippy-js/`; Tailwind `^4.2.4`, Zod `3.25.76`, Vitest `^4.1.4`, Playwright `^1.59.1` (`aicippy/e2e/studio/package.json`).
- CLI: Go `1.25.5` + Cobra, module `github.com/aicippy/cli` (`cli/apps/cli-go/go.mod`); npm shim execs the platform binary.
- Edge Functions: Rust toolchain `1.82.0`, `deno_core` `0.324.0`, `deno_ast` `=0.44.0` (`edge-runtime/`).

## Architecture
Sibling roots under `/Users/aj/Dev-Apps/aicippy-io`; the monorepo proper is `aicippy/`. Paths below relative to it unless prefixed.
- `apps/studio` authenticated console (DB, auth, storage, functions, billing); `apps/www`, `apps/docs` public sites.
- `packages/aicippy-js` (`0.0.0-automated`) isomorphic JS client wrapping `auth-js`, `postgrest-js`, `realtime-js`, `storage-js`, `functions-js`; `packages/pg-meta` Postgres introspection/DDL behind Studio's DB pages.
- `packages/mcp-server-aicippy` MCP tools `search_docs`, `list_tables`, `execute_sql`, `apply_migration`, `get_logs`, `get_advisors`.
- `cli/apps/cli-go` + `cli/apps/cli` Go/Cobra CLI: `start`, `stop`, `link`, `login`, `db`, `migration`, `functions`, `branches`, `secrets`, `gen`, `sso`.
- `edge-runtime/` Rust host embedding Deno; the "user runtime" is isolated behind a "main runtime" proxy.

Flow: the JS SDK reaches a project's Postgrest/Auth/Storage/Realtime/Functions endpoints; the CLI drives local Docker stacks and remote ops; Edge Runtime executes deployed Functions; `e2e/studio` (Playwright) drives Studio, which calls a REST control-plane via `apps/studio/lib/server/platform-api.ts`. Deploy: Vercel, one project per app root (`aicippy-www`, `aicippy-docs`, `aicippy-studio`); Studio also ships a self-hostable Docker image.

## Build / run
Run from `aicippy/`.
| Task | Command |
|---|---|
| Install | `pnpm install` — `preinstall` hook `scripts/require-pnpm.mjs` hard-fails npm/yarn |
| Build | `pnpm build`, or `build:studio` / `build:docs` / `build:design-system` (all set `AICIPPY_TURBO_BUILD=1`, `--concurrency=1`) |
| Dev | `pnpm dev` (parallel) or `dev:studio` / `dev:docs` / `dev:www` |
| Studio on a real local stack | `pnpm dev:studio-local` → `setup:cli` starts the CLI Docker stack, writes `keys.json`, generates local env, runs Studio at `NODE_ENV=test` |
| Gates | `pnpm lint`, `pnpm typecheck`, `pnpm test:prettier` / `pnpm format` |
| Image / E2E | `pnpm build:studio:docker`; `pnpm e2e` in `aicippy/e2e/studio` |

Also required: a running Docker daemon for `setup:cli` and any local stack.

## Non-obvious failure modes
- `edge-runtime` cannot build from crates.io alone: `Cargo.toml` patches `deno_core` → `github.com/aicippy/deno_core` branch `324-aicippy` and `v8` → `github.com/aicippy/rusty_v8` tag `v130.0.7`; git access to both forks is mandatory.
- `packageManager` drifts per root: `aicippy` pnpm `11.8.0`, `cli` pnpm `11.4.0`, `aicippy-js` pnpm `11.1.2`, `storage` npm `11.12.1` — run package commands from the correct root.
- PayPal checkout 400s unless `returnUrl`/`cancelUrl` are on the aicippy.io domain, and 401s without a JWT `sub` claim — both checked before PayPal is called.

## Design system (`DESIGN_SYSTEM.md`, `COMPONENT_REGISTRY.md`)
Not "Supabase green-led", not Liquid Glass. Tokens: `--bg-canvas:#07111f`, `--text-primary:#eef4ff`, `--brand-primary:#5b8cff` (hover `#78a0ff`), `--brand-danger:#ef4444`. 12-col grid, 8px spacing, 1280px max marketing container; buttons never full-width. Shells: `app-shell`→`AppLayout.tsx`, `auth-shell`→`SignInLayout.tsx` (flagged "must become Cognito-facing before launch").

## Naming conventions
- npm scope `@aicippy/*`; platform binaries publish as `@aicippy/cli-<os>-<arch>`.
- Plan codes are Sanskrit: free→`aarambh`, pro→`vajra`, team→`chakra` (`packages/shared-data/plans.ts`); the subscription API zod enum accepts only `vajra`/`chakra`.
- PayPal `custom_id`: `['AICIPPY', userSub, planCode, organizationSlug.slice(0,64)].join('|')`.
- Go CLI commands: lower-kebab-case Cobra `Use:`, e.g. `network-bans`, `vanity-subdomains`.

## Data types & models
- `apps/studio/types/base.ts`, served by the REST Platform API — `Organization{managed_by, partner_id?, plan{id,name}}`, `Project{id:number, ref, status, organization_id:number, region}`, `User{id:number, primary_email, gotrue_id, is_alpha_user:boolean}`.
- `PricingInformation` (`packages/shared-data/plans.ts`): `id`, `platformPlanCode`, `priceMonthly`, `creditsMonthly`.
- PayPal secret (Secrets Manager, zod in `apps/studio/lib/server/paypal.ts`): `clientId, clientSecret, planVajra, planChakra, environment, currency`.
- Central tables `users`, `organizations`, `memberships`, `subscriptions`, `entitlements`, `billing_events`, `audit_events`, `invitations` — prose only in `ARCHITECTURE.md`, store undecided (Aurora v2 or RDS). **No migration or DDL exists**: the only `CREATE TABLE` statements in the tree are pg-meta test fixtures and Docker dev volumes.

## API surface
Routes under `apps/studio/pages/api/platform/billing/paypal/`:
| Method / Path | Request → Response | Auth |
|---|---|---|
| `GET .../config` | none → `{clientId,currency,environment,plans}`, else 503 | none |
| `POST .../subscriptions` | `{organizationSlug,planCode,returnUrl,cancelUrl}` (zod) → `{subscriptionId,status,approvalUrl}` | Bearer JWT |
| `GET .../subscriptions/[subscriptionId]` | path param → PayPal subscription | Bearer JWT |
| `POST .../webhook` | PayPal event + `paypal-*` sig headers → `{received,eventId,eventType,...}` | PayPal sig verify |

MCP tools declare an `inputSchema` and return `CallToolResult`; `aicippy start`/`stop`/`status` drives the local Docker Compose stack.

## Security boundary
- Bearer JWT via `apiWrapper(...,{withAuth:true})` → `apiAuthenticate`. Production verifies with Cognito (`aws-jwt-verify` `CognitoJwtVerifier`, `apps/studio/lib/server/cognito.ts`); non-production falls back to legacy GoTrue claims — transitional dual-auth, not the end state.
- Env vars: browser-exposed use `NEXT_PUBLIC_`, server secrets `AICIPPY_`/`AIVIBE_`; Secrets Manager namespace `aicippy/prod/<service>/<key>`. Names in use: `AICIPPY_PAYPAL_SECRET_ID`, `PAYPAL_WEBHOOK_ID`, `AIVIBE_PAYMENT_API_FUNCTION_NAME`, `VERCEL_TOKEN`, `HCAPTCHA_SITE_KEY`, `POST_HOG_KEY`, `GOOGLE_MAPS_KEY`.
- `www`/`docs` unauthenticated; `studio` authenticated by default; the PayPal webhook public but signature-verified.
- Edge Functions CORS is a shared exported constant — origin `*`, headers `authorization, x-client-info, apikey, content-type, x-retry-count`, methods `GET,POST,PUT,PATCH,DELETE,OPTIONS`; the wildcard is deliberate for user-authored functions.
- Studio headers set globally in `next.config.ts` `headers()`: `X-Frame-Options:DENY`, HSTS (prod on Vercel), dynamic CSP from `apps/studio/csp.ts` allow-listing Cognito/AppSync/PayPal/GitHub/Sentry/hCaptcha; non-platform builds get only `frame-ancestors 'none'`. No CORS on `pages/api/platform/...` — a gap if called cross-origin.

## Known gaps & risks
- **No version control at the project root.** No `.git` at the project root; only independently git-tracked sub-repos (`aicippy/`, `cli/`, `edge-runtime/`, `etl/`, `realtime/`, `storage/`). A change spanning roots cannot be committed atomically.
- AppSync/GraphQL declared in env, CSP and Docker but never called: the only source reference to `NEXT_PUBLIC_APPSYNC_GRAPHQL_URL`/`_REALTIME_URL` is `apps/studio/csp.ts`. Contradicts `ARCHITECTURE.md`'s GraphQL-first directive and the `api.aivibe.cloud` acceptance gate.
- Visual-regression and Lighthouse gates named in `ACCEPTANCE.md` (`e2e/playwright.visual.spec.ts`, `lhci autorun --config=lighthouserc.cjs`) — neither file exists. Cognito cutover incomplete; the 2026-07-09 route inventory still marks many routes "needs audit".
