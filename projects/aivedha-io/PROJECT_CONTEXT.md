# AivedhA.io

## Goal
Aivedha One (aivedha.io) is a SaaS suite bundling AURA (support inbox), ORBIT (scheduling), SEAL (e-signing) under one subscription with shared credits. It runs as product "AIVEDHAIO" on the shared **AiVibe platform**: one multi-tenant Postgres DB (`aivibe_platform`), one Cognito pool, platform-wide billing/subscription/credit tables shared with 8 sibling AiVibe products (incl. the separate product AIVEDHA/aivedha.ai). Target user: small/growing teams wanting unified support, scheduling, e-signing without stitching Chatwoot, Cal.com, Documenso together.

## Core requirements
- Users, tenants, subscriptions, credits live only in shared **platform tables** (`global_users`, `tenants`, `subscription_plans`, `user_subscriptions`, `credit_wallets`, `payment_orders`) — never duplicated locally.
- Product tables (`aura_*`, `orbit_*`, `seal_*`) carry `tenant_id`/`product_id` FKs and Postgres RLS on `app.current_tenant_id` (`...REFERENCE.md` §D; `05_create_rls_policies.sql`).
- SSO across `*.aivedha.io` subdomains via one Cognito PKCE flow, `httpOnly` cookies on `.aivedha.io` (`auth_bridge/routes/auth.ts`).
- Payments are server-authoritative (client picks plan+currency, never amount); webhooks HMAC-verified over raw bytes; settlement idempotent via a monotonic order state machine (`billing-core.ts`).
- Credits shared across AURA/ORBIT/SEAL, tracked per billing cycle (`routes/usage.ts`).

## Tech stack
| Layer | Technology | Version | Source of truth |
|---|---|---|---|
| Monorepo | pnpm workspaces | `pnpm@10.15.1` | `package.json` |
| website / portal | Next.js, React, Tailwind | `next ^15.5.2`, `react ^19.1.1`, `tailwindcss ^4.2.2` | `apps/{website,portal}/package.json` |
| control_api / auth_bridge | Express (Node ESM) | `express 5.1.0` | `apps/{control_api,auth_bridge}/package.json` |
| Payments SDKs | Razorpay, PayPal | `razorpay 2.9.6`, `@paypal/paypal-server-sdk 2.4.0`, `@paypal/react-paypal-js ^8.9.1` | `control_api`, `portal` |
| DB / secrets / JWT | pg, AWS SDK v3, jose | `pg 8.16.0`, `@aws-sdk/client-secrets-manager 3.750.0`, `jose ^6.0.11` | `control_api`, `packages/auth_verify` |
| Language | TypeScript | `^5.9.3` / `5.9.2` | per-app `package.json` |
| Database / Auth | Postgres `aivibe_platform` / Cognito | PG 18.3; pool `us-east-1_S2Cpx3svp` | `db/migrations/final`; `...REFERENCE.md` §E |
| Container / CI | `node:20-alpine`, ECS Fargate, GH Actions Node 20 | — | `infra/`, `.github/workflows/ci.yml` |

## Architecture
Monorepo (`apps/*`, `packages/*`): 4 first-party apps plus 3 externally-deployed forked products sharing the DB/Cognito pool, not sourced here.

| App | Purpose | Path |
|---|---|---|
| website | Marketing site, pricing, product pages, SEO | `apps/website` |
| portal | Authenticated dashboard: billing, checkout, usage, account | `apps/portal` |
| control_api | REST: plans, billing, usage, entitlements | `apps/control_api` |
| auth_bridge | Cognito OIDC/PKCE bridge; issues/renews `.aivedha.io` cookies | `apps/auth_bridge` |
| AURA/ORBIT/SEAL (external) | Chatwoot/Cal.com/Documenso forks, image-deployed | `ecr:aivedha-{aura,orbit,seal}` |

Routing (`docs/launch/aivedha_routing_map.txt`): `aivedha.io`→website, `app.aivedha.io`→portal, `api.aivedha.io`→control_api (+`/auth`→auth_bridge), `{aura,orbit,seal}.aivedha.io`→external products. Portal calls control_api/auth_bridge with `credentials:"include"`; control_api connects **directly to Postgres**, not AppSync. Deploy: Docker→ECR→ECS Fargate (ports 3000/3001/4000/4100) behind an ALB.

## Naming conventions
- App dirs: `snake_case` (`apps/auth_bridge`); packages: scoped kebab-case (`@aivedha/auth-bridge`, `@aivedha/ui`, `@aivedha/auth-verify`).
- DB tables/enums: product-prefixed `snake_case` (`aura_conversations`, `orbit_bookings`, `seal_signing_status`); every table has `tenant_id`/`product_id`/`created_at`/`updated_at`/`deleted_at`.
- REST routes: versioned, e.g. `/api/v1/billing/checkout`. Env vars: `SCREAMING_SNAKE_CASE`, browser vars prefixed `NEXT_PUBLIC_`.
- React: PascalCase; client components suffixed `Client` (`CheckoutClient.tsx`, `UsageClient.tsx`).
- Cookies: `aivedha_id_token`, `aivedha_access_token`, `aivedha_refresh_token`, `aivedha_user`.

## Data types & models
All in shared Postgres `aivibe_platform`.
| Entity | Fields (name : type) | Defined in |
|---|---|---|
| global_users / tenants | id:uuid, vibe_id:text, email:text, cognito_sub:text; tier:TenantTier | `...REFERENCE.md` §G |
| subscription_plans / user_subscriptions | plan_code:text, base_price:numeric, credits_per_cycle:int; status:text, current_period_end:timestamptz | `07_aivedha_io_product_and_plans.sql`, `billing-core.ts` |
| credit_wallets / credit_transactions | balance:numeric; transaction_type:CREDIT\|DEBIT, idempotency_key:text | `billing-core.ts` |
| payment_orders (RLS) | gateway_order_id/payment_id:text, amount:numeric, currency:varchar(3), status:enum | `billing-core.ts`, `08_*.sql` |
| aura_conversations / aura_messages | status:aura_conversation_status; message_type:aura_message_type, content:text | `02_create_aura_tables.sql` |
| orbit_event_types / orbit_bookings | duration_minutes:int; start_at/end_at:timestamptz, status:orbit_booking_status | `03_create_orbit_tables.sql` |
| seal_envelopes / seal_recipients | document_status:seal_document_status; signing_status:seal_signing_status | `04_create_seal_tables.sql` |

## API surface
| Operation | Method / Path | Auth | Defined in |
|---|---|---|---|
| Plans / product modules | GET `/api/v1/plans?cycle=`, `/products/modules` | Public | `routes/plans.ts`, `products.ts` |
| Products / entitlements / tenant / orgs | GET `/api/v1/products`, `/me/{entitlements,tenant,organizations}` | Cognito JWT | `routes/products.ts` |
| Checkout / verify / capture | POST `/billing/checkout` `{planId,gateway,currency?}`, `/verify`, `/paypal/capture` | JWT | `routes/billing.ts` |
| Razorpay / PayPal webhooks | POST `/billing/webhook/{razorpay,paypal}` — raw body + HMAC/transmission-sig | Gateway sig | `routes/billing.ts` |
| Reconcile / subscription / orders / usage | POST `/billing/reconcile`; GET `/me/{subscription,orders,usage,usage/history}` | Cognito JWT | `routes/billing.ts`, `usage.ts` |
| Health (both services) | GET `/health` | Public | `routes/health.ts` |
| OIDC login/callback/me/refresh/logout/token | GET `/auth/{login,callback,me,refresh,logout,token}` | Cookie/Cognito | `auth_bridge/routes/auth.ts` |
| Platform GraphQL (not called here) | AppSync `api.aivibe.cloud/graphql`, 221Q/325M/22S | Cognito/key | `...REFERENCE.md` §F |

`packages/graphql_schema/schema.graphql` is a 2-line health-check stub, not the platform schema.

## CORS & headers
- `control_api`/`auth_bridge`: allowlist CORS (`middleware/cors.ts`) — portal/website/aura/orbit/seal(+api) origins, `Allow-Credentials:true`, `+localhost` in dev.
- Both set headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `Strict-Transport-Security: max-age=31536000; includeSubDomains`.
- Next.js apps set `poweredByHeader:false` only — no `headers()`/CSP — GAP.

## Security boundary
- Auth: Cognito User Pool `us-east-1_S2Cpx3svp` (platform-shared), Authorization-Code+PKCE via `auth_bridge`, id-token verified via `jose` against Cognito JWKS.
- Tenancy: `tenant_id` on every table; product tables enforce Postgres RLS on `app.current_tenant_id`, set per-transaction (`withTenantTransaction`, `db.ts`).
- Secrets (names only, Secrets Manager): `DB_PASSWORD`, `COGNITO_CLIENT_SECRET`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, `PAYPAL_CLIENT_SECRET`, `PAYPAL_WEBHOOK_ID`, `SESSION_SECRET`, `NEXTAUTH_SECRET` — read at runtime via `PAYMENT_SECRET_ID`.
- Public vs private: website/portal shells public; `/me/*` and billing-mutation routes require a Cognito id-token; webhooks authenticate by gateway signature only.

## Known gaps & risks
- **Product-UUID drift (verified):** `control_api/config/env.ts` reads `PRIMARY_PRODUCT_UUID_AIVEDHA`; prod template + CI set it to `bfe08452-…` (product AIVEDHA, **aivedha.ai**). `ENV_MAPPING_REPORT.md` (2026-07-16) says billing must target `af970281-…` (product AIVEDHAIO/aivedha.io, seeded by migration `07`) but names the var `AIVEDHA_PRODUCT_ID`, absent from source/CI — no committed file sets the correct UUID; billing likely targets the sibling aivedha.ai plans.
- **Direct DB access (verified):** platform reference mandates AppSync for platform reads; `control_api` queries Postgres directly via `pg` — no AppSync call exists in `apps/`.
- **Incomplete auth-verify migration (verified):** `packages/auth_verify` claims to replace 3 drifted JWT verifiers; only `portal` uses it — `control_api` checks issuer only, not audience.
- `FINAL_OPEN_ITEMS.md`/`FINAL_STATUS.md` dated 2026-03-31, predate July migrations — historical.
- AURA/ORBIT/SEAL app code is external (image-deployed forks); only their DB schema is verifiable here.
