# VibeKaro Platform

## Goal

VibeKaro (`vibekaro.ai`) turns a natural-language brief into a deployed, production-grade Next.js
or Flutter app. The user briefs agent ArjunA; isolated agent AikuttY builds it in a disposable
Bedrock AgentCore microVM (no ECS/Fargate), streaming ordered events. Projects move through
`requirements → design_development → testing → go_live`, then are hosted on
`<slug>.vibekaro.ai` or a custom domain, with tenant email and subscription/credit billing.

## Tech stack

| Layer | Technology / version (exact pins from manifests) |
| --- | --- |
| Runtime | Node `>=26.5.0`, npm `>=12.0.1`, TypeScript `7.0.2`, oxlint `1.75.0` `--type-aware`, Prettier `3.9.6` |
| Web | Next.js `16.2.12` (`output: "export"`), React `19.2.8`, Tailwind `4.3.3`, Vitest `4.1.10` |
| Contracts / mobile | Zod `4.4.3`, Flutter `3.44.8` |
| IaC / compute | aws-cdk `2.1133.0`, aws-cdk-lib `2.262.1`, `@aws-sdk/*` `3.1095.0`, Lambda `NODEJS_24_X` on `ARM_64` |
| Agents | ArjunA `global.anthropic.claude-sonnet-5`; AikuttY `global.anthropic.claude-opus-5` |
| Data | PostgreSQL via RDS Proxy (TLS-required, instance shared with sibling AiVibe products); DynamoDB (pay-per-request, CMK) |
| Payments | PayPal only: subscriptions + credits (`services/paypal`, `ops/paypal`) |

## Architecture

Browser → CloudFront+WAF → versioned S3 release (static export) and `api.vibekaro.ai` → Lambda
services; Step Functions drives the SDLC through CodeBuild verification and atomic release
promotion; harnesses stream events over AppSync Events (`docs/ARCHITECTURE.md`). Invariants: stage
values identical in contracts, Postgres, and events (`packages/contracts/src/stages.ts`); S3
checkpoints + Git commits are the durable truth, not session state; release is an atomic
S3/CloudFront pointer swap with rollback; credit grants come only from signature-verified webhooks
(append-only ledger).

`services/`: `auth` Cognito PKCE BFF + authorizer; `control` session/catalog/credits/projects/stage
transitions; `workspace` AgentCore broker; `domain` FSM on CloudFront SaaS Manager; `email` SES
tenant + outbox; `generated-app` tenant runtime; `paypal` checkout/webhook/reconciliation; `shared`
RLS helpers. Plus `apps/web`, `packages/contracts`, `agentcore/`, `infra`, `ops/paypal` (bootstrap
CLI), `ops/acceptance` (smoke/cutover).

Three CDK stacks (`infra/bin/vibekaro-platform.ts`), all `terminationProtection: true` (asserted at
synth): `VibeKaroPlatform` + `VibeKaroRelease` in `us-east-1`, `VibeKaroRecovery` in `us-west-2`
(must exist before primary replication). CI `.github/workflows/deploy-infrastructure.yml`:
`workflow_dispatch` only, `ubuntu-24.04`, OIDC (`vars.AWS_DEPLOY_ROLE_ARN`, `id-token: write`);
CodeBuild runs `buildspecs/{web,flutter,execution}.yml` on non-root `containers/` images.

## Build, run, deploy

- Prereqs beyond pinned Node/npm: Docker to publish `containers/` images; local Postgres for
  `npm run database:test`; AWS credentials for `infra:*`.
- `npm ci` → `npm run verify` (deps, contracts, format:check, lint, typecheck, test, database:test,
  coverage, build, npm audit, `infra:synth`). CDK: `npm run infra:deploy`. Deploy is dispatch-only:
  `gh workflow run deploy-infrastructure.yml --ref main`.
- Failure modes: every `apps/web` script `pre*`-builds `@vibekaro/contracts`, so a stale contracts
  build breaks typecheck/test confusingly; `postinstall` runs
  `scripts/reconcile-bundled-dependencies.mjs`, reverting hand-edited `node_modules`; the ~2.30 GB
  Flutter build image has exhausted CI runner disk.

## Naming conventions

- DB tables `public.vibekaro_ai_<entity>_v3`/`_v4`; billing `_v4`
  (`vibekaro_ai_billing_{orders,subscriptions,payments,accounts,price_book,consents}_v4`); shared
  tables in `aivedha.`. Fields snake_case; only telemetry responses use camelCase
  `currentStage`/`progressPercent`. Zod contracts PascalCase + `Schema`.
- Workspaces `@vibekaro/<name>`; AWS resources `vibekaro-prod-<purpose>` (`-http-api`,
  `-paypal-webhooks`, `-migrations`); migrations `NNNN_vibekaro_<slug>.sql`; secrets
  `aivibe/vibekaro/prod/<name>` (names only). Plans `aarambh, raksha, suraksha, vajra, chakra`;
  agents `arjuna, aikutty`.

## Data types & models

Entities: `database/migrations/0001…sql`; billing v4 `0011…sql`.

| Entity | Fields (name : type) |
| --- | --- |
| Project / StageEvent | `project_id:uuid, framework:enum, stage_code:enum` / `sequence_number:bigint, payload:jsonb` |
| CreditWallet / Ledger | `balance:int, reserved:int, transaction_type:enum(9), delta:int, balance_before/after:int` |
| Domain / BillingOrder v4 | `domain_name, status:enum(9)` / `provider:literal('paypal'), amount_minor:bigint` |
| AgentEvent / Plan | `event_type:enum(10)` / `plan_code:enum(5)` (`contracts/agents.ts`, `plans.ts`); Session = opaque DynamoDB record |

## API surface

| Path (ANY method) | Request → Response | Auth | Source (`services/…`) |
| --- | --- | --- | --- |
| `/auth`, `/auth/{proxy+}`, `/api/v1/auth/{proxy+}` | OAuth/PKCE → `Set-Cookie` session | public | `auth` |
| `/api/v1/{session,catalog,credits,projects}[/{proxy+}]` | reserve/debit, file ops, stage transitions → `ApiResponse<...>` | session | `control` |
| `/api/v1/workspace/sessions[/{proxy+}]` | messages → `AgentEvent` | session | `workspace` |
| `/api/v1/billing[/{proxy+}]`, `/api/v1/domains[/{proxy+}]`, `/email/{proxy+}` | `CheckoutCreateInput`/`DomainRegistrationRequest`/template → `CheckoutRedirect`/`DomainBinding` | session | `paypal` |
| `/build/v1/tickets/{id}/{op}`, `/runtime/v1/projects/{project_id}/webhooks/{webhook_key}`, `/webhooks/paypal` | source/artifact, tenant data, raw body+sig → result / 200 | signed | `generated-app`, `paypal` |
| Tools `vibekaro_project_{list,read,write,...}`; AppSync `projects/<id>/<agent_code>`; SQS `*_QUEUE_URL` | tool args / msg → result / `AgentEvent` | AWS_IAM | `agentcore/app/AikuttY/harness.json` |

## CORS, headers, security boundary

CORS (`createHttpApi()`): credentials allowed; request headers `content-type, idempotency-key,
if-match, x-idempotency-key, x-csrf-token, x-request-id`; exposes `x-request-id, retry-after`;
GET/POST/PUT/PATCH/DELETE/OPTIONS; origins `vibekaro.ai` + `www`; 1h preflight. CloudFront
security-headers policy: CSP `connect-src` = `api.vibekaro.ai`, `api-m(.sandbox).paypal.com`,
AppSync HTTPS/WSS; `frame-src`/`form-action` = `www(.sandbox).paypal.com`; `payment=` PayPal only;
`X-Frame-Options: DENY`, HSTS 730d preload, same-origin COOP/CORP. A second policy pins
generated-app `frame-ancestors` to `vibekaro.ai`.

Universal Cognito pool (`auth.aivibe.cloud`) shared across AiVibe products; dedicated VibeKaro
client (PKCE, Cognito+Google, 15-min tokens). The Auth BFF issues only an opaque HttpOnly
`__Host-vibekaro_session` cookie; protected routes are gated by a cookie-derived authorizer. Every
app table is `tenant_id`/`app_id` non-null with forced RLS on
`current_setting('app.tenant_id'/'app.app_id')`, set via `SET LOCAL` per request
(`services/shared/src/database.ts`) under a scoped Postgres role (no BYPASSRLS). Lambdas sit in a
no-ingress VPC with TLS and database-proxy egress only; each has its own role and a scoped
`execute-api:Invoke` grant to `secrets.aivibe.cloud` — app code never reads Secrets Manager.
Env var names (never values): `SECRETS_BROKER_ORIGIN`, `DATABASE_SECRET_ID`,
`DATABASE_PROXY_ENDPOINT`, `DATABASE_STATEMENT_TIMEOUT_MS`, `COGNITO_HOSTED_DOMAIN`,
`AGENT_{MAX_ITERATIONS,TIMEOUT_SECONDS}`, `PAYPAL_{SECRET_ID,WEBHOOK_QUEUE_URL}`, `SES_TENANT_ID`.
Public routes are scoped by ticket/webhook signature.

## Known gaps & risks

1. **Stripe→PayPal migration is complete in code** (verified 2026-08-02): no `stripe` dependency,
   no `vibekaro_ai_stripe_customers_v3`, no Stripe CSP origin. Stripe survives only as immutable
   audit lineage in migrations `0001, 0002, 0008, 0010` and
   `docs/STRIPE_HISTORICAL_MIGRATION_NOTE.md` — never edit or replay them.
2. **Production is behind and unverified** (`docs/PRODUCTION_EXECUTION_STATE.md`).
   `vibekaro-prod-migrations` reports `0001_vibekaro_platform_v3` current; `0002`–`0011` validated
   locally, undeployed. Deploy run `30733760112` (2026-08-02) passed OIDC and role assumption, then
   failed importing the Flutter image (builder out of disk); nothing deployed. The live PayPal
   catalog and webhook endpoint need reconciling without duplicating resources.
3. Postgres engine version is not pinned in CDK; `VibeKaroRecovery` has never been observed live
   despite a hard stack dependency.
