# AivedhA.io

## Goal
Aivedha One (aivedha.io): SaaS suite bundling AURA (support inbox), ORBIT (scheduling), SEAL (e-signing) under one subscription with shared credits. Product **AIVEDHAIO** on the shared AiVibe platform: one multi-tenant Postgres DB (`aivibe_platform`), one Cognito pool, billing tables shared with 9 sibling products — incl. SEPARATE sibling AIVEDHA (aivedha.ai).

## Core requirements
(§ = repo-root `AIVIBE_PLATFORM_INTEGRATION_REFERENCE.md`.)
- Users, tenants, subscriptions, credits live only in shared **platform tables** — never duplicated locally.
- Product tables (`aura_*`, `orbit_*`, `seal_*`) carry `tenant_id`/`product_id` FKs; Postgres RLS on `app.current_tenant_id` (§B). `05_create_rls_policies.sql` covers aura_/orbit_/seal_ ONLY; `payment_orders` RLS is platform-side (§B).
- SSO across `*.aivedha.io` via one Cognito PKCE flow; `httpOnly` cookies on `.aivedha.io` (`aivedha_user` deliberately JS-readable) — `auth_bridge/routes/auth.ts`.
- Payments server-authoritative (client sends `planId`/`gateway`/`currency?`, never amount); webhooks verified over raw bytes; settlement idempotent via monotonic order states CREATED→PENDING→AUTHORIZED→CAPTURED|FAILED|CANCELLED|EXPIRED (`billing-core.ts`).
- Credits shared across AURA/ORBIT/SEAL per billing cycle (`routes/usage.ts`).

## Tech stack (per manifests)
| Layer | Version |
|---|---|
| Monorepo | `pnpm@10.15.1` workspaces |
| website / portal | `next ^15.5.2`, `react ^19.1.1`, `tailwindcss ^4.2.2` |
| control_api / auth_bridge | `express 5.1.0`, Node ESM |
| Payments | `razorpay 2.9.6`, `@paypal/paypal-server-sdk 2.4.0`; portal `@paypal/react-paypal-js ^8.9.1` |
| DB / secrets / JWT | `pg 8.16.0`, `@aws-sdk/client-secrets-manager 3.750.0`, `jose 6.0.11` (auth_verify `^6.0.11`) |
| TypeScript | `5.9.2` services / `^5.9.3` web + packages |
| Database / Auth | Postgres 18.3 `aivibe_platform`; Cognito pool `us-east-1_S2Cpx3svp` |
| Container / CI | `node:20-alpine` on ECS Fargate; GH Actions Node 20 |

## Architecture
| App | Purpose / port |
|---|---|
| `apps/website` | Marketing, pricing, SEO — 3000 |
| `apps/portal` | Dashboard: billing, checkout, usage, account — 3001 |
| `apps/control_api` | REST: plans, billing, usage, entitlements — 4000 |
| `apps/auth_bridge` | Cognito OIDC/PKCE bridge, `.aivedha.io` cookies — 4100 |
| AURA/ORBIT/SEAL | Chatwoot v4.12.1-ce / Cal.com v6.2.0 / Documenso v2.8.1 forks (2026-04-01 audit), image-deployed ECS `aivedha-{aura,orbit,seal}`; `Dockerfile.seal` needs `products/seal/` checkout (absent) |

Routing (`docs/launch/aivedha_routing_map.txt`, ALB rules `DEPLOYMENT_RUNBOOK.md`): `aivedha.io`→website, `app.`→portal, `api.`→control_api, `api.aivedha.io/auth/*`→auth_bridge, `{aura,orbit,seal}.`→forks. Portal calls both services with `credentials:"include"`.

## Build & deploy
- Dev: `pnpm install`; `pnpm dev:{website,portal,control-api,auth-bridge}`; all four: `infra/scripts/run_local_all.sh` (logs in `.aivedha_logs/`) / `stop_local_all.sh`.
- Build: `pnpm build:<app>` — Next `output:"standalone"`; services `tsc` → `node dist/index.js`.
- DB: `FINAL_DB_APPLY.sh` applies 01–06 (needs `PGPASSWORD`); 07 (product+plans) + 08 (indexes) run separately via psql; all idempotent.
- CI (`ci.yml`): PRs, builds only changed apps. Deploy: push to `main` touching that app → `deploy-<app>.yml`: Docker → ECR `aivedha-<svc>` → re-register LIVE task-def, image only → `ecs update-service` → `api.aivedha.io/health` gate.
- **Traps:** runtime env vars exist ONLY in live ECS task-defs — deploys carry them forward; no committed file sets them. `infra/ecs/register-tasks.sh` writes task-defs with ONLY `NODE_ENV`+`PORT` — running it wipes all other env vars; control_api fails boot.

## Naming conventions
- App dirs `snake_case`; packages kebab-case (`@aivedha/auth-verify`). DB tables/enums product-prefixed `snake_case`; tables carry tenant_id/product_id/created_at/updated_at/deleted_at.
- REST routes versioned (`/api/v1/...`); env vars `SCREAMING_SNAKE_CASE`, browser `NEXT_PUBLIC_`. React PascalCase; client components suffixed `Client` (`CheckoutClient.tsx`).
- Cookies: `aivedha_{id,access,refresh}_token`, `aivedha_user`.

## Data types & models
| Entity | Key fields | Source |
|---|---|---|
| global_users / tenants | id:uuid, vibe_id:text, email:text, cognito_sub:text; tier:TenantTier | §G |
| subscription_plans / user_subscriptions | plan_code:text, base_price:numeric, credits_per_cycle:int; status:SubscriptionStatus, current_period_end:timestamp | §G, `07_*.sql` |
| credit_wallets / credit_transactions | balance:numeric; transaction_type:CREDIT\|DEBIT, idempotency_key:text | `billing-core.ts` |
| payment_orders | gateway_order_id/payment_id:varchar(200), amount:numeric, currency:varchar(3), status:payment_order_status | §G, `billing-core.ts` |
| aura_conversations/messages, orbit_event_types/bookings, seal_envelopes/recipients | product enums: aura_conversation_status, aura_message_type, orbit_booking_status, seal_document_status, seal_signing_status | `01–04_*.sql` |

## API surface
| Operation | Method / Path | Auth |
|---|---|---|
| Plans / modules | GET `/api/v1/plans?cycle=MONTHLY\|YEARLY`, `/products/modules` | Public |
| Products / me | GET `/api/v1/products`, `/me/{entitlements,tenant,organizations}` | Cognito JWT |
| Checkout / verify / capture | POST `/billing/checkout` `{planId,gateway,currency?}`, `/verify`, `/paypal/capture` | JWT |
| Webhooks | POST `/billing/webhook/{razorpay,paypal}` (sig over raw body) | Gateway sig |
| Reconcile / usage | POST `/billing/reconcile`; GET `/me/{subscription,orders,usage,usage/history}` | Cognito JWT |
| Health | GET `/health` (auth_bridge also `/auth/health`) | Public |
| OIDC (auth_bridge) | GET `/auth/{login,callback,me,refresh,logout,token}` | Cookie/Cognito |
| Platform GraphQL (unused here) | AppSync `api.aivibe.cloud/graphql`, 221Q/325M/22S (§F) | Cognito/key |

`packages/graphql_schema/schema.graphql` is a 2-line health stub, not the platform schema.

## Security boundary
- CORS allowlist + credentials (`middleware/cors.ts`): control_api allows portal/website/aura/orbit/seal origins, auth_bridge also api, +localhost in dev. Both set nosniff, `X-Frame-Options: DENY`, Referrer-Policy, HSTS. Next.js apps: `poweredByHeader:false` only — no `headers()`/CSP — GAP.
- auth_bridge (`lib/jwks.ts`) verifies issuer+token_use, audience only when client id set; portal via `@aivedha/auth-verify` (issuer+audience+token_use, fails closed without client id).
- Tenancy: `withTenantTransaction` (`db.ts`) sets `app.current_tenant_id` via transaction-local `set_config`.
- Payment secrets: Secrets Manager JSON at `PAYMENT_SECRET_ID` (default path SHARED with sibling AiPohA — `env.ts`) — keys `razorpay_key_id|key_secret|webhook_secret`, `paypal_client_id|client_secret|webhook_id`, `paypal_mode`; same-name env fallbacks; gateway enabled only if id+secret resolve. Plus envs `DB_PASSWORD`, `COGNITO_CLIENT_SECRET`, `SESSION_SECRET`.
- `/me/*` + billing mutations need a Cognito id-token (Bearer or `aivedha_id_token` cookie); webhooks authenticate by gateway signature only.

## Known gaps & risks
- **Product-UUID drift (verified):** code reads `PRIMARY_PRODUCT_UUID_AIVEDHA` (`config/env.ts`); the ONLY committed value (`config/env/.env.production.final.template`) is `bfe08452-…` = sibling AIVEDHA/**aivedha.ai**. `ENV_MAPPING_REPORT.md` (2026-07-16) orders the switch to AIVEDHAIO but names var `AIVEDHA_PRODUCT_ID`, which no code reads; CI never sets env vars. Fix: set `PRIMARY_PRODUCT_UUID_AIVEDHA=af970281-ed30-43d2-a606-e8468239f785` (products.AIVEDHAIO, migration 07) on the live control_api task-def AND template — else plans/billing serve aivedha.ai.
- **Direct platform-table access (verified):** reference §D allows direct DB only for product-prefixed tables (platform reads via AppSync); §H forbids direct platform writes; control_api reads AND WRITES `payment_orders`/`user_subscriptions`/`credit_wallets`/`credit_transactions` (`billing-core.ts`).
- **control_api JWT under-validation (verified):** `lib/auth.ts` checks issuer only — no audience/token_use. Any shared-pool token (sibling-product clients, access tokens) can call billing APIs. `@aivedha/auth-verify` fixes this; only portal migrated.
- `docs/operations/FINAL_*` reports (2026-03-31/04-01) predate July 2026 migrations 07/08 — historical only.
- AURA/ORBIT/SEAL app code is external; only their DB schema is verifiable here.
