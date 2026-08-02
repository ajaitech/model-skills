# VibeKaro Platform

## Goal

VibeKaro (`vibekaro.ai`) converts a natural-language brief into a deployed, production-grade
Next.js or Flutter app. The user briefs user-facing agent ArjunA; a second, isolated agent
(AikuttY) builds it in a disposable Bedrock AgentCore microVM, streaming ordered progress events.
Every project moves through four stages — `requirements → design_development → testing →
go_live` — then is hosted on `<slug>.vibekaro.ai` or a custom domain, with tenant email and
subscription/credit billing built in. Audience: individuals and small teams shipping a production
app without hand-writing code.

## Core requirements

- Exactly four canonical stage values enforced identically in contracts, Postgres, and events.
- ArjunA is the only user-facing chat surface; AikuttY streams implementation events separately,
  in an isolated Bedrock AgentCore microVM per session — no ECS/Fargate.
- S3 checkpoints + Git commits are the durable source of truth, not agent session state.
- Every app table is `tenant_id`/`app_id` non-null with forced Postgres RLS, scoped via
  `SET LOCAL app.tenant_id`/`app.app_id` per request.
- Auth BFF issues only an opaque, HttpOnly session cookie; Cognito tokens never reach the browser.
- Production release is an atomic S3/CloudFront pointer swap with rollback; credit grants are
  authoritative only from signature-verified webhooks (append-only ledger).
- GitHub Actions authenticates to AWS solely via OIDC; no static AWS keys.

## Tech stack

| Layer | Technology | Version | Source of truth |
| --- | --- | --- | --- |
| Runtime / web / UI / lang | Node.js / npm / Next.js (SSG) / React / TypeScript | `>=26.5.0` / `>=12.0.1` / `16.2.12` / `19.2.8` / `7.0.2` | `package.json`, `apps/web` |
| CSS / validation / test / mobile | Tailwind / Zod / Vitest / Playwright / Flutter | `4.3.3` / `4.4.3` / `4.1.10` / `1.62.0` / `3.44.8` | `apps/web`, `contracts`, `ops/acceptance`, `containers/flutter` |
| IaC / SDK / compute | CDK / aws-cdk-lib / `@aws-sdk/*` / Lambda Node 24 arm64 | `2.1133.0` / `2.262.1` / `3.1095.0` | `infra/package.json` |
| Agent runtime | Bedrock AgentCore Harness | Sonnet 5 / Opus 5 | `agentcore/app/*/harness.json` |
| Data stores | PostgreSQL (shared, RDS Proxy) + DynamoDB | Postgres version unpinned — unverified | `infra/src/config.ts` |
| Payments (mid-migration) | `stripe` SDK, still wired despite rename | `22.3.2` | `ops/paypal/package.json` |

## Architecture

Browser → CloudFront+WAF → versioned S3 release (Next.js SSG) and `api.vibekaro.ai` → Lambda
services for auth, control, workspace, billing, domain, email, generated-app; Step Functions drives
the four-stage SDLC through CodeBuild verification and atomic release promotion. AgentCore
Harnesses stream ordered events over AppSync Events. (`docs/ARCHITECTURE.md`)

| Service / App | Purpose | Path |
| --- | --- | --- |
| Web | Marketing site + Studio shell (Next.js) | `apps/web` |
| Auth | Cognito PKCE BFF, session issuance, authorizer | `services/auth` |
| Control | Session, catalog, credits, projects, stage transitions | `services/control` |
| Workspace | AgentCore broker: sessions/events, realtime authorizer | `services/workspace` |
| Domain / Email | Domain FSM (CloudFront SaaS Mgr); SES tenant + outbox | `services/domain`, `services/email` |
| Generated-app | Runtime for tenant apps: data/jobs/webhooks/auth | `services/generated-app` |
| PayPal (was Stripe) | Checkout + webhook, still Stripe-typed | `services/paypal` |
| Shared/Contracts/AgentCore | RLS DB helpers; Zod contracts; agent harnesses | `services/shared`, `packages/contracts`, `agentcore/` |
| Infra / Ops | CDK stacks; billing bootstrap CLI; prod smoke/cutover | `infra`, `ops/paypal`, `ops/acceptance` |

CI/CD: GitHub Actions → AWS via OIDC only. CodeBuild runs `web.yml`/`flutter.yml` (scan, lint,
typecheck, tests, E2E, a11y, Lighthouse, SBOM, release) and `execution.yml` (tenant preview) on
non-root `containers/` images. Target: CDK `VibeKaroPlatform`, account `<AWS_ACCOUNT_ID>`,
`us-east-1`, cross-region `VibeKaroRecovery` in `us-west-2`, both `terminationProtection: true`.

## Naming conventions

- DB tables: `public.vibekaro_ai_<entity>_v3`/`_v4`; shared tables in `aivedha.`
  (`aivedha.sessions`, `aivedha.paypal_webhook_events_v4`).
- Fields: snake_case everywhere; only telemetry responses use camelCase `currentStage`/
  `progressPercent`. Zod contracts: PascalCase + `Schema` suffix (`CheckoutCreateInputSchema`).
- npm workspaces: `@vibekaro/<name>`. AWS resources: `vibekaro-prod-<purpose>`
  (`vibekaro-prod-stripe-webhook` — not yet renamed).
- Plan codes: `aarambh, raksha, suraksha, vajra, chakra`; agent codes: `arjuna, aikutty`.
  Migrations: `NNNN_vibekaro_<slug>.sql`. Secrets: `aivibe/vibekaro/prod/<name>` (names only).

## Data types & models

| Entity | Fields (name : type) | Store | Defined in |
| --- | --- | --- | --- |
| Project, StageEvent | `project_id:uuid, framework:enum, stage_code:enum` / `sequence_number:bigint, payload:jsonb` | PostgreSQL | `0001…sql` |
| CreditWallet/Ledger | `balance:int, reserved:int, transaction_type:enum(9), delta:int, balance_before/after:int` | PostgreSQL | `0001…sql` |
| Domain, BillingAccount/Order (v4) | `domain_name, status:enum(9)` / `provider:literal('paypal'), amount_minor:bigint` | PostgreSQL | `0001…sql`, `0011…sql` (unapplied) |
| AgentEvent, Plan, Session | `event_type:enum(10)` / `plan_code:enum(5)` / opaque RLS record | PostgreSQL+AppSync, DynamoDB | `contracts/agents.ts`, `plans.ts` |

## API surface

| Operation | Method / Path | Request shape | Response shape | Auth | Defined in |
| --- | --- | --- | --- | --- | --- |
| Auth flow | ANY `/auth`, `/auth/{proxy+}`, `/api/v1/auth/{proxy+}` | OAuth/PKCE params | `Set-Cookie` session | public | `services/auth/src/handler.ts` |
| Session/Catalog/Credits/Projects/Workspace | ANY `/api/v1/{session,catalog,credits,projects}[/{proxy+}]`, `.../workspace/sessions[/{proxy+}]` | reserve/debit, file ops, stage transitions, messages | `ApiResponse<...>`/`AgentEvent` | session | `services/control`, `services/workspace` |
| Billing / Domains / Email | ANY `/api/v1/billing[/{proxy+}]`, `.../domains[/{proxy+}]`, `/email/{proxy+}` | `CheckoutCreateInput` / `DomainRegistrationRequest` / template fields | `CheckoutRedirect`/`DomainBinding`/`ApiResponse` | session | `services/paypal`, `platform-stack.ts` |
| Build ticket / generated runtime / webhooks | `/build/v1/tickets/{id}/{op}`, `/runtime/v1/projects/{id}/{proxy+}`, `.../webhooks/{key}`, `/webhooks/stripe` (unchanged) | source/artifact; tenant data; raw body+sig | result / passthrough / 200 | signed | `services/generated-app`, `paypal` |
| AgentCore tools + realtime + queues | `vibekaro_project_{list,read,write,...}`; AppSync `projects/<id>/<agent_code>`; SQS `*_QUEUE_URL` | tool args; queue msg | tool result / `AgentEvent` stream | AWS_IAM | `agentcore/app/AikuttY/harness.json` |

## CORS & headers

HTTP API CORS (`createHttpApi()`): credentials allowed; headers `content-type, idempotency-key,
if-match, x-idempotency-key, x-csrf-token, x-request-id`; methods GET/POST/PUT/PATCH/DELETE/
OPTIONS; origins restricted to `vibekaro.ai`/`www`; 1h preflight cache. CloudFront policy
`vibekaro-prod-security-headers`: CSP still allows `*.stripe.com` (Stripe-specific despite the
PayPal migration), `X-Frame-Options: DENY`, HSTS (730d, preload), same-origin
`Cross-Origin-Opener/Resource-Policy`. A second policy restricts generated-app embedding to
`frame-ancestors vibekaro.ai`.

## Security boundary

Single universal Cognito pool (`auth.aivibe.cloud`), shared across AiVibe products, with a
dedicated VibeKaro app client (PKCE, Cognito+Google, 15-min tokens). The Auth BFF issues only an
opaque, HttpOnly `__Host-vibekaro_session` cookie; Cognito tokens never reach the browser, and
every protected route is gated by a cookie-derived Lambda authorizer. Every app table is
`tenant_id`/`app_id` non-null with forced RLS filtered on
`current_setting('app.tenant_id'/'app.app_id')`, set via `SET LOCAL` per request
(`services/shared/src/database.ts`) under a scoped Postgres role (no BYPASSRLS), on an instance
shared with sibling AiVibe products. Each Lambda has its own role and a scoped
`execute-api:Invoke` grant to `secrets.aivibe.cloud` — app code never reads Secrets Manager
directly. Env var names observed (never values): `SECRETS_BROKER_ORIGIN`, `DATABASE_SECRET_ID`,
`COGNITO_HOSTED_DOMAIN`, `AGENT_MAX_ITERATIONS`, `STRIPE_WEBHOOK_QUEUE_URL`. Secret root:
`aivibe/vibekaro/prod/` (names only). Public routes (auth, billing catalog, tickets, generated
webhooks) are scoped by ticket/webhook signature; GitHub CI reaches AWS solely via OIDC.

## Known gaps & risks

1. **Stripe→PayPal migration is live and uncommitted mid-repo.** `services/stripe`→`services/paypal`
   and `ops/stripe`→`ops/paypal` are renamed and `_v4` billing tables/contracts exist, but
   `checkout-handler.ts`/`webhook-handler.ts` still import the `stripe` SDK, query
   `vibekaro_ai_stripe_customers_v3`, and the webhook route is still `/webhooks/stripe`.
   `ops/paypal/package.json` is still named `@vibekaro/stripe-bootstrap`; CDK, the CSP, and
   `README.md` are untouched.
2. **Production is behind and unverified.** Applied migration is `0001_vibekaro_platform_v3`
   only — `0002`–`0011` are unapplied; GitHub Actions run `30478369252` failed at
   `sts:AssumeRoleWithWebIdentity`, so CDK deploy/migration/smoke never ran for the current commit
   (`docs/PRODUCTION_EXECUTION_STATE.md`). The only known PayPal credential parsed as live (no
   sandbox validated); `VibeKaroRecovery` (us-west-2) was not observed live despite a hard CDK
   dependency; Postgres engine version is unpinned.
