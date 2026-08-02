# Aicippy.io

## Goal
AiCippy is a production backend-as-a-service platform and public marketing/docs site, rebranded from a Supabase-derived code estate into a fully AiCippy-owned surface: managed Postgres, auth, storage, realtime, and edge-functions via a web console ("Studio"), a CLI, and isomorphic JS SDKs, monetized with PayPal subscriptions (`Aarambh`/free, `Vajra`/pro, `Chakra`/team). Audience: developers on AiCippy-hosted backends, and the AiCippy team running `aicippy.io`/`docs.aicippy.io`/`studio.aicippy.io`.

## Core requirements
From `ACCEPTANCE.md`/code:
- Zero residual `supabase`, `supa-`, `stripe` strings in owned code/docs/routes.
- Build/lint/typecheck/tests/e2e/visual-regression/Lighthouse CI must pass; the latter two (`playwright.visual.spec.ts`, `lhci autorun`) are **not yet configured**.
- PayPal is the only payment provider; no Stripe/card-field references.
- `auth.aivibe.cloud` (Cognito) and `api.aivibe.cloud` (AppSync) must be reachable; authz enforced server-side.
- Secrets only via AWS Secrets Manager under `aicippy/prod/...` namespaces; one Vercel project per app root.

## Tech stack
| Layer | Technology | Version (exact) | Source of truth |
|---|---|---|---|
| Monorepo tooling | pnpm workspaces + Turborepo + Node.js | pnpm `11.8.0`, turbo `2.9.14`, Node `>=24 <25` | `aicippy/package.json`, `.nvmrc` |
| Web framework | Next.js (studio, www) | `16.2.10` | `pnpm-workspace.yaml` catalog |
| UI runtime | React / react-dom | `19.2.7` | `pnpm-workspace.yaml` catalog |
| Language | TypeScript | `~5.9.3` (studio/www); `~5.8.3` (aicippy-js) | `pnpm-workspace.yaml` |
| CSS / validation / test | Tailwind `^4.2.4`, Zod `3.25.76`, Playwright `^1.59.1`, Vitest `^4.1.4` | — | `pnpm-workspace.yaml`, `e2e/studio/package.json` |
| CLI | Go `1.25.5` (Cobra, `github.com/aicippy/cli`) + Node/Bun shim `aicippy` `0.0.0-automated` | — | `cli/apps/cli-go/go.mod` |
| Edge Functions runtime | Rust host embedding Deno core; Rust `1.82.0`, `deno_core` `0.324.0`, V8 tag `v130.0.7` | — | `edge-runtime/Cargo.toml` |
| JS/TS client SDK | `@aicippy/aicippy-js` (wraps auth/postgrest/realtime/storage/functions-js) | `0.0.0-automated` | `packages/aicippy-js/package.json` |

## Architecture
Five sibling code roots; monorepo root is `aicippy/` (per `ARCHITECTURE.md`).

| Package | Purpose | Path |
|---|---|---|
| `studio` | Authenticated console (DB, auth, storage, functions, billing) | `aicippy/apps/studio` |
| `www` / `docs` | Public marketing / docs sites | `apps/www`, `apps/docs` |
| `@aicippy/aicippy-js` | Isomorphic JS client (auth/postgrest/realtime/storage/functions) | `packages/*-js` |
| `@aicippy/mcp-server-aicippy` | MCP tools (`search_docs`,`list_tables`,`execute_sql`,`apply_migration`,`get_logs`,`get_advisors`) | `packages/mcp-server-aicippy` |
| `cli-go`+`cli` | Go/Cobra CLI (`start`,`stop`,`link`,`login`,`db`,`migration`,`functions`,`branches`,`secrets`,`gen`,`sso`) wrapped by npm shim that execs the platform binary | `cli/apps/cli-go`, `cli/apps/cli` |
| Edge Runtime | Rust host embedding Deno; "user runtime" isolated behind "main runtime" proxy | `edge-runtime/` |

Connection: the JS SDK is what apps (and Studio) use to call a project's Postgrest/Auth/Storage/Realtime/Functions endpoints; the CLI drives local Docker dev stacks and remote ops; Edge Runtime executes deployed Functions; `e2e/studio` (Playwright) drives Studio behaviorally. Studio talks to a REST platform control-plane (`lib/server/platform-api.ts`). `ARCHITECTURE.md` targets GraphQL-over-AppSync + Cognito, but `NEXT_PUBLIC_APPSYNC_GRAPHQL_URL` is wired only into CSP/Docker (no GraphQL call site found); Cognito is used only for JWT verification.

Deployment: Vercel, one project per app root (`aicippy-www`, `aicippy-docs`, `aicippy-studio`); Studio also ships a self-hostable Docker image.

## Design system
From `DESIGN_SYSTEM.md` / `COMPONENT_REGISTRY.md`:
| Aspect | Rule |
|---|---|
| Brand intent | Premium, precise, technical, calm under complexity; explicitly not "Supabase green-led" |
| Core tokens | `--bg-canvas:#07111f`, `--text-primary:#eef4ff`, `--brand-primary:#5b8cff`, `--brand-danger:#ef4444` |
| Layout | 12-col grid, 8px spacing, max 1280px marketing container |
| Components | Buttons: medium radius, never full-width; pricing cards: one primary plan emphasis only |
| Canonical | `app-shell`→`AppLayout.tsx`; `auth-shell`→`SignInLayout.tsx` (flagged "must become Cognito-facing before launch") |

## Naming conventions
- npm scope `@aicippy/*`, e.g. `"@aicippy/mcp-server-aicippy"`, `"@aicippy/cli-darwin-arm64"`.
- Plan codes are Sanskrit, not English tiers: `tier_free`→`aarambh`, `tier_pro`→`vajra`, team→`chakra`.
- PayPal `custom_id` composite key: `['AICIPPY', userSub, planCode, organizationSlug].join('|')`.
- Env vars: browser-exposed `NEXT_PUBLIC_`; server secrets `AICIPPY_`/`AIVIBE_` prefix; Secrets Manager namespace `aicippy/prod/<service>/<key>` e.g. `aicippy/prod/paypal/client-id`.
- Go CLI commands: lower-kebab-case Cobra `Use:` values, e.g. `"network-bans"`, `"vanity-subdomains"`.

## Data types & models
| Entity | Fields (name : type) | Store | Defined in |
|---|---|---|---|
| `Organization` | `managed_by:ManagedBy`, `partner_id?:string`, `plan:{id:PlanId,name:string}` | Platform API (REST) | `apps/studio/types/base.ts` |
| `Project` | `id:number`,`ref:string`,`status:string`,`organization_id:number`,`region:string` | Platform API (REST) | same file |
| `User` | `id:number`,`primary_email:string`,`gotrue_id:string`,`is_alpha_user:boolean` | Platform API (REST) | same file |
| `PricingInformation` | `id`,`platformPlanCode`,`priceMonthly`,`creditsMonthly` | Static config | `packages/shared-data/plans.ts` |
| `PayPalSecret` | `clientId,clientSecret,planVajra,planChakra,environment,currency` (zod) | Secrets Manager | `apps/studio/lib/server/paypal.ts` |
| Central tables (`users`,`organizations`,`memberships`,`subscriptions`,`entitlements`,`billing_events`,`audit_events`,`invitations`) | Described only | Aurora v2 or RDS (undecided) | `ARCHITECTURE.md` — **unverified, no migration found** |

## API surface
| Operation | Method / Path / CLI | Request | Response | Auth | Defined in |
|---|---|---|---|---|---|
| PayPal billing config | `GET .../paypal/config` | none | `{clientId,currency,environment,plans}` or 503 | None | `paypal/config.ts` |
| Create PayPal subscription | `POST .../paypal/subscriptions` | `{organizationSlug,planCode,returnUrl,cancelUrl}` (zod) | `{subscriptionId,status,approvalUrl}` | Bearer JWT | `.../subscriptions.ts` |
| PayPal webhook | `POST .../paypal/webhook` | PayPal event + `paypal-*` sig headers | `{received,eventId,eventType,...}` | PayPal signature verify | `.../webhook.ts` |
| CLI local stack | `aicippy start`/`stop`/`status` | Docker Compose | container status | Local Docker | `cmd/start.go` |
| MCP tools | `search_docs`,`list_tables`,`execute_sql`,`get_logs` | zod per tool | MCP `CallToolResult` | Platform creds | `mcp-server-aicippy/src/index.ts` |

## CORS & headers
- Edge Functions CORS is a shared exported constant: `Access-Control-Allow-Origin:'*'`, headers `authorization, x-client-info, apikey, content-type, x-retry-count`, methods `GET,POST,PUT,PATCH,DELETE,OPTIONS` — `@aicippy/aicippy-js/cors`. Wildcard by design for user Edge Functions.
- Studio sets security headers globally (`next.config.ts` `headers()`): `X-Frame-Options:DENY`, HSTS (prod+Vercel), dynamic CSP from `csp.ts` allow-listing Cognito/AppSync/PayPal/GitHub/Sentry/hCaptcha; non-platform builds get `frame-ancestors 'none'` only. No CORS found on `pages/api/platform/...` — GAP if called cross-origin.

## Security boundary
- Auth: Bearer JWT via `apiWrapper(...,{withAuth:true})`→`apiAuthenticate`. Production verifies via AWS Cognito (`aws-jwt-verify`'s `CognitoJwtVerifier`); non-production falls back to legacy GoTrue claims — transitional dual-auth state.
- Secret names only (no values): `AICIPPY_PAYPAL_SECRET_ID`, `PAYPAL_WEBHOOK_ID`, `AIVIBE_PAYMENT_API_FUNCTION_NAME`, `VERCEL_TOKEN`, `HCAPTCHA_SITE_KEY`, `POST_HOG_KEY`, `GOOGLE_MAPS_KEY`.
- Public vs private: `www`/`docs` unauthenticated; `studio` authenticated by default; PayPal webhook public but signature-verified.

## Known gaps & risks
- **No version control at the project root.** `/Users/aj/Dev-Apps/aicippy-io` has no `.git`; it's a loose set of independently git-tracked sub-repos (`aicippy/`, `cli/`, `edge-runtime/`, plus out-of-scope `etl/`, `realtime/`, `storage/`), no single history.
- AppSync GraphQL is declared in env/CSP/Docker but not called from any Studio app code found — contradicts the GraphQL-first directive.
- Central relational schema is prose-only in `ARCHITECTURE.md`; no migration/DDL exists in read scope.
- Visual-regression/Lighthouse CI gates not yet configured; Cognito cutover incomplete (non-prod uses GoTrue fallback); route inventory (Jul 9 2026) lists many routes "needs audit"; pnpm drifts across roots (`11.8.0`/`11.4.0`/`11.1.2`).
