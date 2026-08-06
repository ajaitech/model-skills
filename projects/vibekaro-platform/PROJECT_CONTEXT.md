# VibeKaro Platform

_Verified against live code + live AWS (account `936668162296`) 2026-08-07. Supersedes the 2026-08-02 version wholesale — ~130 commits and 7 migrations landed between the two verifications._

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
| Web | Next.js `16.2.12` (`output: "export"`, `trailingSlash: true`), React `19.2.8`, Tailwind `4.3.3` (imported once in `globals.css`; a bespoke class vocabulary does the actual styling — see Design system), Vitest `4.1.10` |
| Contracts / mobile | Zod `4.4.3`, Flutter `3.44.8` |
| IaC / compute | aws-cdk `2.1133.0`, aws-cdk-lib `2.262.1`, Lambda `NODEJS_24_X` on `ARM_64` |
| Agents | ArjunA + AikuttY on Bedrock AgentCore — **model IDs pinned down from Claude 5, see Known gaps #1** |
| Data | PostgreSQL via RDS Proxy onto an **imported** existing instance `aivibe-production-db` (no `rds.DatabaseInstance` construct exists in this repo — engine-version pinning is not possible from CDK here); DynamoDB (pay-per-request, CMK) |
| Payments | PayPal only: subscriptions + credits (`services/paypal`, `ops/paypal`). Stripe fully removed from code; frozen audit lineage only |
| Frontend auth client | `amazon-cognito-identity-js 6.3.20` — the only npm `dependencies` entry in `apps/web` |

## Architecture

Browser → CloudFront+WAF → versioned S3 release (static export) and `api.vibekaro.ai` → Lambda
services; Step Functions drives the SDLC through CodeBuild verification and atomic release
promotion; harnesses stream events over AppSync Events. Invariants: stage values identical in
contracts, Postgres, and events (`packages/contracts/src/stages.ts`); S3 checkpoints + Git commits
are the durable truth, not session state; release is an atomic S3/CloudFront pointer swap with
rollback; credit grants come only from signature-verified webhooks (append-only ledger).

**`services/` is 9 deployable Lambda services, not 8** — `migrations` is a real service, not just a
build step:

| Service | Actual responsibility today |
| --- | --- |
| `auth` | Cognito PKCE BFF **and** direct in-page sign-in exchange (`POST /v1/auth/session` verifies a client-proved Cognito id_token against pool JWKS, mints the same opaque cookie); tenant-claim reconciliation back into Cognito; support-ticket proxy to the global AiVibe GraphQL API; authorizer serving both API Gateway and AppSync Events |
| `control` | Session/catalog/studio-overview/projects CRUD/stage transitions; `/v1/credits` is **GET-only** now (mutation lives in DB functions); also the project-deletion orchestrator (SQS worker, domain-worker invoke, AgentCore session stops, credit refunds) |
| `workspace` | Project-scoped workspace HTTP broker **and** the SQS FIFO dual-agent worker (Bedrock AgentCore `InvokeHarnessCommand`) **and** the SDLC/CodeBuild executor **and** the public build-ticket redemption endpoint (`/build/v1/tickets/{id}/...`) |
| `domain` | Project-scoped domain reserve/verify/refresh/teardown on CloudFront SaaS Manager; atomic release-artifact publishing; EventBridge outbox drain |
| `email` | Tenant domains/senders/templates/messages, **first-party OTP** (`/v1/email/otp/request`, `/otp/verify`), 12 transactional template codes, SES event-stream delivery/suppression processing |
| `generated-app` | Tenant runtime data/routes/jobs/webhooks API; management manifest/evidence/executions/preview API; SSRF-guarded signed outbound egress; **and its own complete end-user Cognito PKCE auth stack** per generated app (`/__vibekaro/auth/*` via CloudFront path `/__vibekaro/*`) |
| `paypal` | Checkout (9 routes incl. capture/portal/subscription lifecycle), webhook verification + credit grants, reconciliation polling. `GET /v1/billing/catalog` is deliberately public; PayPal API origin is hardcoded live (`api-m.paypal.com`) |
| `migrations` | `{migrate\|status\|publish_catalog\|verify_catalog}` — applies `database/migrations/*.sql` one-transaction-per-file under an advisory lock, **and** publishes/verifies the PayPal price catalog under a separate advisory lock |
| `shared` | 16 modules — config, crypto, database (RLS), http, idempotency, session, secrets, errors, retry, logging, agent-event, aws-http, sdlc, project-source, project-teardown, generated-backend. Far beyond "RLS helpers" |

Plus `apps/web` (reorganised 2026-08-04 into `src/{design,ui,api,validation,state,seo}` — `app/`
now holds only route entry points; `apps/web/components` and `apps/web/lib` are empty untracked
shells, safe to ignore), `packages/contracts`, `agentcore/`, `infra`, `ops/paypal` (bootstrap CLI:
`verify-catalog`, `verify-webhook` — catalog **publishing** is not in this CLI, it's the migrations
Lambda's `publish_catalog` action), `ops/acceptance` (Playwright smoke/cutover, `@vibekaro/production-acceptance`).

Two real CDK stacks deployed (verified live via `aws cloudformation list-stacks`, both
`UPDATE_COMPLETE` 2026-08-04): `VibeKaroPlatform` and `VibeKaroRelease`, both `us-east-1`,
`terminationProtection: true`. **`VibeKaroRecovery` (us-west-2) is declared in `infra/bin/vibekaro-platform.ts`
but was never deployed — confirmed zero stacks in us-west-2.** Its primary-side counterpart,
`infra/src/recovery-controls.ts` (backup vault, daily backup plan, cross-region S3 replication,
CloudTrail), was never instantiated by any stack and was **deleted from source 2026-08-07** as
dead code — cross-region recovery needs a fresh design, not a resurrection of that file. CI:
`.github/workflows/deploy-production.yml` (renamed from `deploy-infrastructure.yml` 2026-08-03),
`workflow_dispatch` only, `ubuntu-24.04`, OIDC; CodeBuild runs `buildspecs/{web,flutter,execution}.yml`
on non-root `containers/` images built by a dedicated AWS CodeBuild image builder (not GitHub runners — see CI/CD).

## Design system (frontend)

Not a component library import — Tailwind 4.3.3 is imported once (`@import 'tailwindcss'`) at the
top of a 5,499-line `src/design/globals.css`; **zero Tailwind utility classes appear in TSX**. All
styling is a bespoke class vocabulary (`liquid-glass`, `interactive--{tone|size}`,
`login-dialog__*`, `state-panel`, `bottom-dock`, `scene`) driven by `--*-rgb` custom properties
themed via `:root[data-theme]` + `prefers-color-scheme` fallback. Three primitives in
`src/design/primitives`: `GlassSurface` (depth `near|mid|far`, computes composited background
contrast via a shared `MutationObserver` on `data-theme`), `Interactive` (link-or-button, 4 tones,
3 sizes), `JsonLd`. The visual standard is **executable**, not aspirational:
`src/ui/shell/visual-contract.test.ts` asserts the fixed glass dock, no sub-10px radii, pointer-gated
3-channel hover inversion + 3D lift, `perspective:1100px` scene, and `prefers-reduced-motion`
handling. Shell: `RootLayout → AppProviders → SiteShell` = skip-link, `BackgroundScene`, logo-orbit
header, `Breadcrumbs` top-rail, `ProfileOrbit` (the single floating bottom-right account control —
greets anonymous, morphs to profile menu with theme cycling + logout), `BottomDock`, `LegalFooter`,
`LoginModal`.

## Sign-in — rebuilt 2026-08-05/06, in code but **not yet deployed**

Sign-in no longer redirects to the Cognito hosted UI. A six-screen `LoginModal`
(signin/signup/verify/forgot/reset/one-time-code) authenticates **in-page** via
`amazon-cognito-identity-js`: password uses Cognito SRP, one-time-code uses `CUSTOM_AUTH`. The
proved Cognito `id_token` is POSTed to `/api/v1/auth/session`, which verifies it against the pool
JWKS server-side and mints the same opaque `__Host-vibekaro_session` cookie — the token itself
never round-trips further. Google and GitHub remain as a PKCE popup flow (480×640, state stored in
`localStorage 'vibekaro_social_pkce'`, outcome reported via both `postMessage` and
`BroadcastChannel 'vibekaro-social'` to survive COOP severing the opener) completing at
`/auth/callback`. `github` is gated by `GITHUB_AUTH_ENABLED`.

**One account per proven email** (migration `0018`, commits `3fb2a56`/`47e30f9`/`a7d37f9`):
`aivedha.bootstrap_vibekaro_user` looks up by `cognito_sub` first, then falls back to
`lower(email)` — password, one-time-code, and social identities that prove the same verified email
now resolve to **one account**, which keeps the first identity it saw. This also fixed a `42702`
plpgsql column-ambiguity bug and dropped a stale insert into the Stripe-era (now legacy-schema)
v3 subscriptions table. 0018 is written, locally tested, and wrapped in its own transaction — but
**not applied to production** (DB is at 0017; the 2026-08-04 deploy predates it). Applying it
requires either the next green deploy or a documented one-off owner-run
`aws lambda update-function-code` + `{"action":"migrate"}` invoke (`docs/requirements-ledger.md`).

CDK no longer creates its own `UserPoolClient` — it **adopts the shared AiVibe SSO client**
`2gj7kdplhfg4aoenqfjghpotie` by ID (`infra/src/config.ts`), because a platform-created client had
no Managed Login branding and served a 403. **The currently-deployed `VibeKaroPlatform` stack (last
updated 2026-08-04, before this adoption commit) still owns 1 `AWS::Cognito::UserPoolClient`
resource** (confirmed via `list-stack-resources`) — the old, brandingless, duplicate client. The
next successful deploy removes it; deleting it out-of-band first would strand the CFN record.

## Build, run, deploy

- Prereqs beyond pinned Node/npm: Docker to publish `containers/` images; local Postgres for
  `npm run database:test` (self-provisions a throwaway cluster via `initdb`/`pg_ctl`); AWS
  credentials for `infra:*`.
- `npm ci` → `npm run verify`: `dependencies:reconcile → contracts:build → format:check → lint →
  typecheck → test → database:test → test:coverage → build → audit:dependencies → infra:synth`.
  Deploy is dispatch-only: `gh workflow run deploy-production.yml --ref main`.
- `apps/web` itself has **no lint/format scripts** — those are root-only (`oxlint --type-aware`,
  `prettier`). Every `apps/web` script still pre-builds `@vibekaro/contracts`.
- Flutter image disk exhaustion (the old blocker) is **resolved**: image builds moved entirely off
  GitHub runners onto a dedicated AWS CodeBuild `X2_LARGE` privileged builder
  (`infra/src/release-stack.ts`, `ReleaseImageBuilder`) with S3 digest-memoized results — GitHub
  runners no longer build container images at all.

## Naming conventions

- DB tables: **not uniformly `_v3`/`_v4`** — roughly 10 core tables carry no version suffix
  (`vibekaro_ai_projects`, `project_files`, `stage_events`, `workspace_sessions`,
  `workspace_commands`, `agent_events`, `prompt_drafts`, `idempotency_records`, `catalog_versions`,
  `phase_signoffs`). Billing is `_v4` (9 public tables + `aivedha.paypal_webhook_events_v4`).
  Stripe-era tables were renamed and moved to schema `aivedha_billing_legacy_audit.legacy_*_v3`
  (read-only, `vibekaro_system` only). Shared tables live in `aivedha.*`.
- AWS resources: `vibekaro-prod-<purpose>` for everything current. Migrations
  `NNNN_vibekaro_<slug>.sql`. Secrets by name only (see AWS resources below). Plans `aarambh,
  raksha, suraksha, vajra, chakra` (only `vajra`/`chakra` are subscribable; `aarambh` = free tier =
  **absence** of a v4 subscription row). PayPal lookup-key prefix is now `gpt_vibekaro_` (`vibekaro_`
  is historical-only). Agents `arjuna, aikutty`.

## Data types & models

Migration ledger is **0001 through 0018** (not 0011). Post-2026-08-02 additions:

| # | Purpose |
| --- | --- |
| 0012 | PayPal webhook `provider_environment='live'` boundary |
| 0013 | `app_version` semver release-identity stamping on `app_registry_v3` |
| 0014 | Price-book legacy-key removal + active-price partial index |
| 0015 | Drop one specific leftover prod constraint |
| 0016 | Catalog-identity unique index |
| 0017 | Retire that index — content-addressed catalog republish needs to reinsert historical rows |
| 0018 | Bootstrap disambiguation — 42702 fix, explicit ids, one-account-per-proven-email (see Sign-in) |

Billing v4 is **9 public tables**, not 6: `billing_{accounts,price_book,orders,payments,consents,
refunds,disputes,subscriptions,reconciliation}` + `aivedha.paypal_webhook_events_v4`. RLS since
migration `0005` is **`owner_isolation`** (three settings: `app.tenant_id`, `app.app_id`,
`app.user_id`, plus a project-ownership `EXISTS` subquery), not plain tenant/app isolation — and
`0005` revoked runtime access from 7 legacy unscoped tables.

| Entity | Fields (name : type) |
| --- | --- |
| Project / StageEvent | `project_id:uuid, framework:enum, stage_code:enum` / `sequence_number:bigint, payload:jsonb` |
| CreditWallet / Ledger | `balance:int, reserved:int, transaction_type:enum(9), delta:int, balance_before/after:int` |
| Domain / BillingOrder v4 | `domain_name, status:enum(9)` / `provider:literal('paypal'), amount_minor:bigint` |
| AgentEvent / Plan | `event_type:enum(10)` / `plan_code:enum(5)`; Session = opaque DynamoDB record |

## API surface

The actual CDK route table (`infra/src/platform-stack.ts`) has ~24 routes. Corrections vs the old
table: there is **no `/api/v1/domains`** — domains are project-scoped
`/api/v1/projects/{project_id}/domains[/{proxy+}]`, served by `domain`, not `paypal`. Email is
`/api/v1/email/{proxy+}`, not `/email/{proxy+}`. Workspace is project-scoped
`/api/v1/projects/{project_id}/workspace/sessions[/{session_id}/(messages|events|realtime-token|state)]`.

| Path (ANY method) | Auth | Source |
| --- | --- | --- |
| `/auth`, `/auth/{proxy+}`, `/api/v1/auth/{proxy+}`, `GET /auth/generated/callback` | public | `auth` |
| `GET /api/v1/billing/catalog`, `GET/PUT /build/v1/tickets/{id}/{op}`, `POST /runtime/v1/projects/{id}/webhooks/{key}`, `/webhooks/paypal` | public | `paypal`, `workspace`, `generated-app` |
| `/api/v1/{session,catalog,studio/overview,credits[/proxy],projects[/proxy]}`, `/api/v1/projects/active` | session cookie | `control` |
| `/api/v1/projects/{id}/workspace/sessions[/proxy]` | session cookie | `workspace` |
| `/api/v1/billing/{proxy+}`, `/api/v1/projects/{id}/domains[/proxy]`, `/api/v1/email/{proxy+}` | session cookie | `paypal`, `domain`, `email` |
| `/api/v1/projects/{id}/{backend-manifest,evidence/{id},executions/{proxy+}}`, `/runtime/v1/projects/{id}/{proxy+}` | session / signed | `generated-app` |

Every protected `ANY` route gets an extra unauthenticated `OPTIONS` shadow route so CORS preflight
bypasses the cookie authorizer — this is deliberate, not a gap.

## CORS, headers, security boundary

CORS: credentials allowed; origins `vibekaro.ai` + `www`; 1h preflight. Two CloudFront
`ResponseHeadersPolicy`: `vibekaro-prod-security-headers` (CSP `connect-src` =
`api.vibekaro.ai`, PayPal, AppSync HTTPS/WSS; **`unsafe-inline` script-src, documented Next.js
static-export hydration rationale, hash-pinning not yet in place**; `COOP: same-origin-allow-popups`
specifically to keep the social sign-in popup alive) and
`vibekaro-prod-generated-app-security-headers` (pins generated-app `frame-ancestors` to
`vibekaro.ai`). `X-Frame-Options: DENY`, HSTS 730d preload.

Sessions: **two** cookies — `__Host-vibekaro_session` (opaque, HMAC-hashed server-side, HttpOnly
Secure SameSite=Lax, 14d default / 30d cap) and a short-lived `__Host-vibekaro_auth_flow` PKCE-flow
cookie (10 min) bound to a DB row in `aivedha.auth_flows`. Every app table is
`tenant_id`/`app_id`/`user_id` non-null with `FORCE RLS` (`owner_isolation` since 0005), set via
`SET LOCAL` per request under a scoped Postgres role. Lambdas sit in a no-ingress VPC.

**Secrets Manager access is not uniformly brokered** — this is a correction, not a footnote:
`services/shared/src/secrets.ts` reads Secrets Manager **directly** for the PayPal secret and for
the database secret (via a direct-secret-id mapping); every other secret goes through the
SigV4-signed secrets-broker (`secrets.aivibe.cloud`) with retry + 5-min cache. `COGNITO_HOSTED_DOMAIN`
does not exist as an env var anywhere in `services/*` — the Cognito domain
(`auth.aivibe.cloud`) is a hardcoded constant in `services/auth/src/config.ts`, alongside the
**hardcoded** pool ID `us-east-1_S2Cpx3svp` in two source files (`services/auth/src/handler.ts`,
`services/generated-app/src/auth.ts`) — flagged as a mandate violation (hardcoded ID where a
variable belongs) worth fixing, not yet fixed.

## AWS resources — verified live, account `936668162296`, 2026-08-07

Two real, deployed stacks (`aws cloudformation list-stacks`), both `us-east-1`,
`UPDATE_COMPLETE 2026-08-04`:

- **`VibeKaroPlatform`** — 18 Lambda functions, 3 API Gateway v2 routes tables (41 routes / 25
  integrations / 1 authorizer), 1 AppSync Events API + 1 channel namespace, 3 DynamoDB tables, 1
  RDS Proxy + target group (onto the imported `aivibe-production-db` instance — no owned DB
  instance), 5 S3 buckets + bucket policies, 4 KMS keys + aliases, 2 WAFv2 Web ACLs, 2
  BedrockAgentCore harnesses, 1 Step Functions state machine, 4 CodeBuild projects, 1 SES
  configuration set, 1 SNS topic, 23 CloudWatch alarms, 27 IAM roles / 33 IAM policies, 7 EventBridge
  rules, 6 Lambda event-source mappings (SQS), 3 CloudFront functions, 1 CloudFront distribution
  (the SaaS Manager tenant distribution) + 1 connection group + 2 origin access controls + 2
  response-headers policies, **1 Cognito UserPoolClient (stale — see Sign-in)**.
- **`VibeKaroRelease`** — 1 S3 bucket + policy, 1 CodeBuild project (`ReleaseImageBuilder`,
  privileged, X2_LARGE), 1 IAM role/policy, 1 log group.
- **`VibeKaroRecovery` (us-west-2) does not exist.** Zero stacks in that region — never deployed.
  Its intended primary-side counterpart (`recovery-controls.ts`) was never wired into any stack and
  has since been deleted from source (see "Legacy/orphaned resources" below).

**Resources deliberately imported, not owned by CDK** (confirmed by cross-referencing
`infra/src/config.ts` + `platform-stack.ts` against live AWS): shared VPC `vpc-092ffcb81e4853354`
and database VPC `vpc-0966934b56930347a`; RDS instance `aivibe-production-db`; security groups
`sg-044e721f00925ff0b` / `sg-0b066c92000acc1cf`; Cognito pool `us-east-1_S2Cpx3svp`
(`aivibe-users-production`) + the shared SSO client; Route53 zones for `aivibe.cloud` and
`vibekaro.ai` (**`Z10373692DTPKAOWHPAH5`**, 21 records — CDK creates **no** Route53 records itself,
`domain_worker` and the deploy role mutate records at runtime via IAM grants only); ACM certs;
existing secrets-broker API `6vo3nd19rg`; existing main CloudFront distribution **`E2Q9XRD49M5F29`**
(alias `vibekaro.ai` + `*.vibekaro.ai`, comment "VibeKaro User Webapp Deployments — Wildcard
*.vibekaro.ai" — CDK only tests/publishes an unpublished viewer-request function onto it, never
mutates the distribution itself); SES tenant `vibekaro-prod`; cross-account ECR
`977916686374:repository/harness-us-east-1` (managed AgentCore harness image).

**All 17 SQS queues and all DLQs are imported by ARN (`existing=true`), not created by CDK** — this
is deliberate retained-resource adoption (2026-08-03), not a gap. Verified live names:
`vibekaro-prod-{domain-events, domain-events-dlq, email-events-dlq, email-outbox.fifo,
email-outbox-dlq.fifo, generated-jobs, generated-jobs-dlq, paypal-webhooks.fifo,
paypal-webhooks-dlq.fifo, project-deletions.fifo, project-deletions-dlq.fifo, workspace-events,
workspace-events-dlq, workspace-events.fifo, workspace-events-dlq.fifo}`.

**Live AgentCore runtimes** (`bedrock-agentcore-control list-agent-runtimes`, both `READY`,
version 37): `harness_vibekaro_prod_arjuna`, `harness_vibekaro_prod_aikutty`.

**Live Lambda naming**: 15 functions match the current CDK source exactly —
`vibekaro-prod-{auth-bff, auth-authorizer, realtime-authorizer, control-api, workspace-broker,
paypal-checkout, paypal-webhook, paypal-reconciliation, domain-worker, email-api, email-worker,
migrations, generated-app-runtime, agentcore-runtime-id, transaction-search-handler}`.

**Live DynamoDB**: `vibekaro-prod-{auth-sessions, domains, idempotency}` — matches source exactly.

**Live S3, current/CDK-owned**: `vibekaro-prod-{artifacts, ci, projects, releases,
storage-access, web}-936668162296-us-east-1`.

**Live ECR, current/CI-created (not CDK-declared, referenced by digest-pinned CfnParameters)**:
`vibekaro-prod-build-web`, `vibekaro-prod-build-flutter`.

### Legacy/orphaned resources — verified dead, then deleted 2026-08-07

Every item below was checked for real usage first (message counts, invocation metrics, IAM
`RoleLastUsed`, secret `LastAccessedDate`, image push dates) before deletion, not deleted on name
match alone — a name-match-only pass would have wrongly nuked the ECS cluster `vibekaro-workspaces`,
which turned out to host a **different, still-live sibling product** (`aivibe-cms`), not this one.
That cluster and its `aivibe-cms` service were left untouched.

**Deleted, us-east-1** (all confirmed 0 messages / 0 invocations-30d / never-used / stale-pushed
before deletion):
- SQS: `vibekaro-prod-workspace-events` + `-dlq` (dead standard-queue leftover; the FIFO pair with
  per-session message groups was the one `infra/src/platform-stack.ts:556` actually imported),
  `vibekaro-prod-stripe-webhooks.fifo` + `-dlq.fifo` (no `stripe` code anywhere post the 2026-08-02
  PayPal migration)
- ECR: `vibekaro-build-web`, `vibekaro-build-flutter` (no `-prod-` prefix, superseded by the
  `-prod-` repos CI actually publishes to), `vibekaro-flutter-workspace` (last pushed 2026-01-09,
  7 months stale)
- Lambda: `vibekaro-memory-reaper`, `vibekaro-appsync-resolver`, `vibekaro-workspace-idle-reaper`,
  `vibekaro-workspace-manager`, `vibekaro-arjuna-agent`, `vibekaro-razorpay-webhook` — none appear
  in current `infra/src`; `razorpay-webhook` was doubly stale since Razorpay was never this
  project's billing provider (PayPal is, Stripe was before that)
- IAM roles (9, detached/deleted): `vibekaro-ecs-task-execution-role`, `vibekaro-workspace-task-role`,
  `vibekaro-workspace-manager-lambda-role`, `vibekaro-stepfunctions-role`,
  `vibekaro-lambda-execution-role`, `vibekaro-kb-role`, `vibekaro-appsync-rds-role`,
  `vibekaro-codebuild-flutter-role`, `codebuild-vibekaro-docker-role` — the ECS-flavored names
  pointed to an earlier Fargate-based prototype predating the current AgentCore-microVM
  architecture (whose goal statement explicitly says "no ECS/Fargate")
- Secrets Manager: `aivibe/vibekaro/prod/stripe`

**Deleted, ap-south-1** (the account's default CLI region — a second sweep this project's own code
never touches; found because the account default region silently accumulated stragglers):
- ECR: `vibekaro-golden-workspace`
- Secrets Manager: `vibekaro/paypal-credentials` — a **second, wrong-region duplicate** of the one
  live PayPal secret (`aivedha.ai/payments/paypal`, us-east-1, referenced by `infra/src/config.ts`)

**Retained on explicit instruction** (found stale but kept): `vibekaro/gemini-api-key`,
`vibekaro/agents/config`, `vibekaro/payments/razorpay`.

**Renamed, not deleted** (S3 bucket names can't be re-pointed in place; contents preserved under a
`legacy-` prefix since S3 bucket names disallow underscores): `vibekaro-agent-knowledge` →
`legacy-vibekaro-agent-knowledge`, `vibekaro-webapp-deployments` → `legacy-vibekaro-webapp-deployments`.

**Current, correct, single source of truth going forward**: 27 CDK-auto-named
`VibeKaroPlatform-*ServiceRole*` IAM roles + 5 explicitly-named `vibekaro-prod-{agentcore-execution,
appsync-events-logs, database-proxy, github-deploy, release-image-builder}`; `aivedha.ai/payments/paypal`
+ `aivibe/vibekaro/prod/session` secrets; `vibekaro-prod-{build-web,build-flutter}` ECR repos;
`vibekaro-prod-{workspace-events,paypal-webhooks}.fifo` + matching `-dlq.fifo` SQS pairs (FIFO only).

**Repo-level duplicates also resolved 2026-08-07**: `infra/src/recovery-controls.ts` (939 lines,
never imported by any stack) deleted from source; `apps/web/components/*` and `apps/web/lib/*`
(empty untracked shells left over from the 2026-08-04 `src/` reorg) removed from disk.

**Still-live code-level duplicates, not yet resolved** (same fact hardcoded/repeated in more than
one place instead of a single source):
- Cognito pool ID `us-east-1_S2Cpx3svp` hardcoded literally in both `services/auth/src/handler.ts`
  and `services/generated-app/src/auth.ts`.
- Agent model IDs disagree in three places: `infra/src/config.ts` (real, deployed —
  `claude-sonnet-4-6`/`claude-opus-4-6-v1`) vs `agentcore/app/{ArjunA,AikuttY}/harness.json` (stale:
  `claude-sonnet-5`/`claude-opus-5`) vs `docs/ARCHITECTURE.md` (repeats the stale claim).
  `infra/src/config.ts` is the only correct one.
- The deployed `VibeKaroPlatform` stack (last updated 2026-08-04) still owns 1 duplicate
  `AWS::Cognito::UserPoolClient` alongside the newly-adopted shared SSO client — removed only by
  the next successful deploy, not yet run (see CI/CD).
- Database secret has two IDs for the same credential (`DATABASE_SECRET_ID` via broker,
  `DATABASE_DIRECT_SECRET_ID` direct-read bypass) in `services/shared/src/secrets.ts`.

## AgentCore

Two `bedrockAgentCore.CfnHarness` (ArjunA, AikuttY): VPC network mode, session storage
`/mnt/workspace`, idle 900s / max lifetime 14400s, memory disabled, `converse_stream`, temperature
0.1, model maxTokens 32768, harness maxTokens 65536, maxIterations 64, timeout 840s, 48-message
sliding-window truncation. ArjunA's `allowedTools` is `["@communication"]` only. AikuttY has 12
inline tools: `vibekaro_project_{list,read,write,delete,diff,checkpoint,build}`,
`vibekaro_backend_manifest_{put,read}`, `vibekaro_current_docs_lookup`, `vibekaro_project_execute`,
`vibekaro_project_execution_status`.

## CI/CD — actual current state, not the deploy-run-30733760112 story

The old Flutter-disk-exhaustion blocker is resolved (image builds moved to a dedicated AWS
CodeBuild builder). The current blocker is different: **~48 dispatched `deploy-production.yml`
runs, zero fully green** (46 failure, 2 cancelled). Production is nevertheless **live** — the last
successful infra+web-release publish was run `30954723969` (sha `246c338c`, 2026-08-04), whose
authenticated-acceptance job failed at the AppSync Events cutover step, and whose auto-rollback
also failed (`scripts/restore-main-release.sh` has a known `NoSuchVersion` defect — it fails
closed, so the newer release stays live rather than getting rolled back badly). So
`https://vibekaro.ai` currently serves the `246c338c` web build against the Aug-4 infrastructure.

The very next dispatch (`30958601780`, sha `246ac6e`) failed differently and **deterministically
will fail again**: the deploy role is denied
`cognito-idp:DescribeManagedLoginBrandingByClient` at the new "prove the hosted sign-in page is
served" gate, and no CDK code grants that permission anywhere. This needs a grant added to
`GitHubDeployRole` (`infra/src/platform-stack.ts`) before the next dispatch can get past that step.

14 commits (`2fd16be`..`a7d37f9`, 2026-08-05/06 — the entire in-product sign-in rework, the shared
SSO-client adoption, migration 0018, and the sign-in design passes) are **written, pushed, and
completely undeployed**.

Gate chain: `authorize (main-only) → verify (reusable) → infrastructure (180min) → web_release
(reusable) → authenticated_acceptance (360min, Playwright) → legacy_cleanup (acceptance-gated)`,
plus a conditional `rollback_failed_acceptance`. `verify.yml` layers CodeQL (local zero-alert SARIF
gate; upload disabled — Team plan has no Code Security) and CycloneDX SBOMs on top of the
unchanged root `npm run verify` chain.

## Known gaps & risks

1. **Agent models are pinned off Claude 5, not on it.** Since commit `060c074` (2026-08-03), ArjunA
   runs `global.anthropic.claude-sonnet-4-6` and AikuttY runs `global.anthropic.claude-opus-4-6-v1`
   — Bedrock refuses Claude 5 models for this account (no entitlement; needs an owner action to
   enable, tracked in the commit message as revert-when-granted). `agentcore/app/*/harness.json` and
   `docs/ARCHITECTURE.md` still say `claude-sonnet-5`/`claude-opus-5` — those files are themselves
   stale and contradict their own declared source of truth (`infra/src/config.ts`).
2. **`VibeKaroRecovery` was never deployed**, and its primary-side backup/replication/CloudTrail
   construct (`recovery-controls.ts`) was dead code, now removed. Cross-region recovery does not
   exist today in any form — needs a fresh design, not a resurrection of the deleted file.
3. **Sign-in fix is undeployed.** In-product sign-in, the shared-SSO-client adoption, and migration
   0018 are all committed but blocked behind the Cognito IAM gate above. Until deployed, production
   still runs the old redirect flow against the stale duplicate UserPoolClient.
4. **Migration 0018 is unapplied** (DB at 0017) — requires the fixed deploy or a documented one-off
   manual Lambda code update + invoke.
5. **`scripts/restore-main-release.sh` has a live `NoSuchVersion` defect** — automatic rollback on a
   failed release currently cannot succeed; it fails closed (leaves the new release live) rather
   than corrupting the site, but this is not the intended behavior.
6. Hardcoded Cognito pool ID (`us-east-1_S2Cpx3svp`) in two `services/*` source files, and a CSP
   `unsafe-inline` script-src for Next.js static-export hydration (hash-pinning not yet in place) —
   both flagged, neither fixed.
7. Legacy/duplicate AWS resources from an earlier (ECS-based) architecture generation are still
   live and unreferenced by current code — see the AWS resources section above for the full,
   verified list. Not deleted; flagged for a deliberate cleanup pass.
