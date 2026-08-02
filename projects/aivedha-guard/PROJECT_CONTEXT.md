# AivedhA Guard

## Goal

AiVedha Guard (npm `aivedha-guard`, live at `aivedha.ai`) is an AI-powered website security
audit platform. A user submits a URL; the backend runs a 41-step scan (SSL/TLS, DNS,
headers, XSS/SQLi/SSRF/XXE/JWT detection, AI risk synthesis via "AICIPPY AI"), streams live
progress, and produces a PDF report, a score/grade, and — above a threshold — a public
certificate + badge. Users: website owners, security teams, startups; GitHub
Marketplace/Actions lets CI/CD trigger audits. Monetized via plans (`aarambh` free through
`chakra` enterprise) plus credit packs, billed via Razorpay (PayPal behind a flag). One
product in the AiVibe SaaS suite, sharing Cognito SSO and one RDS.

## Core requirements

- Audit lifecycle is atomic: URL validate → credit check → scan → live progress (AppSync
  sub) → all items → PDF → certificate/badge if qualified → email → **credit deducted only
  after the email succeeds** (EXECUTION_MANDATE §3). Findings never redacted, incl. found
  secrets (§2).
- Universal data (users/plans/subscriptions/credits/billing) owned by `api.aivibe.cloud` /
  shared `aivibe_platform` RDS; app data stays in `aivedha_ai_*`-prefixed tables only.
- Zero hardcoded plan codes/prices/URLs/emails — config/env/DB sourced; `FALLBACK ONLY`
  fixtures labeled. Types agree case-sensitively across `audit.types.ts` ↔ `schema.graphql`
  ↔ Lambda dicts.
- Razorpay is default gateway; PayPal only if `VITE_ENABLE_PAYPAL=true`. Secrets load via
  `secrets_loader.py` (Secrets Manager, cached, env fallback), never hardcoded.

## Tech stack

| Layer | Technology | Version | Source of truth |
|---|---|---|---|
| Frontend | React+TS, Vite+SWC, Tailwind/shadcn, react-router/react-query | 18.3.1/5.9.3/7.3.1/3.4.19 | `package.json` |
| API (primary) | AWS AppSync GraphQL | API `sgig66576fhyvc2or2beq775ne` | `aws-appsync/schema.graphql` |
| API (fallback) | API Gateway + Lambda REST | `https://api.aivedha.ai/api` | `src/lib/api.ts:91` |
| Backend | AWS Lambda, Python (py3.12 target) | — | `deploy/deploy_lambda.sh:76` |
| Database | PostgreSQL, shared `aivibe_platform` RDS | — | `shared/db_connection.py` |
| Auth | Cognito Hosted UI, shared AiVibe pool | `us-east-1_S2Cpx3svp` | `src/lib/cognito.ts:3` |
| Hosting | S3 + CloudFront | `aivedha-website-936668162296` | `deploy/deploy_frontend.sh` |
| Payments | Razorpay SDK (Python) | `razorpay==1.4.2` | `razorpay-handler/requirements.txt` |

App version `2.7.4` = `API_VERSION`. No `aws-amplify` — hand-rolled `graphql-ws` client.

## Architecture

Client-only Vite/React SPA on S3+CloudFront. Two backend paths coexist: AppSync GraphQL
fronting Lambda resolvers with a native-WebSocket `onAuditProgress` subscription for live
progress; and a legacy REST path via API Gateway (`src/lib/api.ts`) slated for retirement.
~55 Lambdas live under `aws-lambda/<name>/`, deployed as `aivedha-guardian-<short-name>`
via `deploy/deploy_lambda.sh`, sharing a common layer + copied `shared/` modules. DB
access is direct psycopg2 (`shared/db_connection.py`) against one shared Postgres
instance; app tables use `aivedha_ai_*`, while `shared/universal_db.py` runs direct SQL
against universal tables (see gaps). Frontend deploys via CI on push to `main`; Lambda
deploys are manual/scripted.

## Naming conventions

- Components PascalCase, one/file (`Dashboard.tsx`); other TS modules kebab-case
  (`appsync-client.ts`); Lambda dirs kebab-case → `aivedha-guardian-<short-name>`.
- DB tables prefixed `aivedha_ai_` (collision-proofing in shared `public` schema);
  columns snake_case.
- GraphQL types PascalCase; fields dual-cased — camelCase canonical (`progressPercent`)
  plus snake_case legacy alias (`progress_percent`) on one type (`schema.graphql:198`).
- Env vars: frontend `VITE_*`; backend upper-snake (`RDS_HOST`,`RAZORPAY_KEY_ID`).
- Plan codes: Sanskrit-derived — `aarambh`,`raksha`,`suraksha`,`vajra`,`chakra`.
- Branches `<type>/<desc>-<YYYY-MM-DD>` (e.g. `fix/production-hardening-pass-2026-04-27`).
  `APP_ID`=`'aivedha-guard'` matches `gateway_config.py` `AIVIBE_APPS` key.

## Data types & models

Rows in `database/aivedha_ai_schema.sql` (line noted).

| Entity | Key fields | Store | Line |
|---|---|---|---|
| Audit report | `report_id PK`,`user_id:UUID`,`status`,`security_score:NUMERIC(4,2)`,`grade`,`credit_used:BOOL`,`scan_results:JSONB`,`items_total default 41` | `aivedha_ai_audit_reports` | 35 |
| Audit finding | `finding_id:UUID PK`,`report_id FK`,`severity`,`evidence:TEXT (never redacted)`,`cvss_score`,`ai_status(pending\|done\|reused\|failed)` | `aivedha_ai_audit_findings` | 124 |
| Certificate | `certificate_number PK`,`report_id FK`,`security_score`,`grade`,`revoked:BOOL` | `aivedha_ai_certificates` | 169 |
| Scheduled audit | `schedule_id:UUID PK`,`cron_expr`,`active:BOOL`,`next_run_at` | `aivedha_ai_scheduled_audits` | 195 |
| Addon purchase | `purchase_id:UUID PK`,`addon_code(scheduler\|credits_*)`,`provider` | `aivedha_ai_addon_purchases` | 352 |
| Ticket/blog | `ticket_id PK`,`blog_id`,`slug UNIQUE`,`rating 1-5` | Postgres | 254 |

## API surface

Schema: 24 Query/42 Mutation/1 Subscription fields. Frontend prefers GraphQL
(`appsync-api.ts`), REST fallback (`api.ts`). 55 Lambda `lambda_handler` entries.

| Operation | Type | Auth | schema.graphql line |
|---|---|---|---|
| `startAudit`/`getAuditStatus`/`startCicdAudit` | Mutation/Query | API key / per-user `X-API-Key` | 1156, 1063, 1209 |
| `onAuditProgress`/`publishAuditProgress` | Sub/Mutation | api_key+iam / iam-only | 1255, 1124 |
| `authenticateUser`/`registerUser`/`googleAuth`/`authenticateGitHub` | Mutation | API key | 1163-1167 |
| `createRazorpayOrder`/`Subscription`/`verifyPayment`, `createPayPalOrder`+2 | Mutation | API key | 1183-1191 |
| `createApiKey`/`deleteApiKey`/`revokeApiKey` | Mutation | `@aws_cognito_user_pools` | 1203-1205 |
| `handleGitHubWebhook` | Mutation | HMAC (`GITHUB_WEBHOOK_SECRET`) | `github-auth/lambda_function.py` |

## CORS & headers

Centralized in `aws-lambda/shared/cors.py`; allow-list via `CORS_ALLOWED_ORIGINS`, default
`https://aivedha.ai,https://www.aivedha.ai,https://admin.aivedha.ai`.
`resolve_origin()` echoes request Origin only if allow-listed; emits
`Access-Control-Allow-Credentials: true`, methods `GET,POST,PUT,DELETE,OPTIONS`. Applies to
REST Lambda responses; AppSync handles its own CORS.

## Security boundary

- Frontend auth: Cognito Hosted UI, shared pool `us-east-1_S2Cpx3svp` at `auth.aivibe.cloud`.
  GraphQL: AppSync `API_KEY` default + `AWS_IAM` extra provider; some fields
  `@aws_cognito_user_pools`.
- Webhooks: GitHub HMAC (`GITHUB_WEBHOOK_SECRET`); Razorpay/PayPal secrets
  `RAZORPAY_WEBHOOK_SECRET`, `PAYPAL_WEBHOOK_ID`.
- Secrets: AWS Secrets Manager `aivibe/*` via `secrets_loader.py` (10-min cache, env
  fallback). Names only: `aivibe/razorpay/key_id`, `aivibe/jwt/admin_signing_key`,
  `aivibe/production/database`, `RDS_HOST`, `RAZORPAY_KEY_ID`, `ADMIN_JWT_SECRET`.
- Public: marketing, `/pricing`, `/certificate/:num`, `/verify/:num`, blog, badges. Private
  (`<PrivateRoute>`): `/dashboard`, `/security-audit`, `/purchase`, `/profile`, `/scheduler`.
  `/admin/*` uses a separate admin-JWT flow. Findings (incl. secrets found) show to the
  report owner unredacted, by design.

## Known gaps & risks

- **IaC vs schema mismatch**: schema header claims "ALL operations served through
  AppSync," but the CFN template defines only 2 resolvers (`publishAuditProgress`
  passthrough + placeholder Query). ~65 other fields' resolvers are managed imperatively
  via `deploy/update_appsync_resolvers.py`, outside checked-in IaC.
- **Universal-data boundary violated by design**: docs say universal tables go only via
  `api.aivibe.cloud`; `shared/universal_db.py` runs direct SQL against them instead (no
  AppSync M2M auth yet, per its comment). `MIGRATION_STATUS.md`'s referenced
  `shared/universal_api.py` no longer exists — docs stale vs code.
- **README stale**: advertises "21 modules, 178+ checks" — crawler actually runs a fixed
  41-item/8-phase pipeline (`security-audit-crawler.py:1672`). Also documents PayPal only;
  real default is Razorpay (`ENABLE_PAYPAL` false); version badge lags `package.json`.
  Raksha/Suraksha plans ship with empty Razorpay IDs (`INFRA_RUNBOOK.md` R3).
- **TLS misconfig (P0)**: `api.aivedha.ai` serves the execute-api default cert, not the
  `*.aivedha.ai` ACM cert — breaks direct API/CI-CD consumers (`INFRA_RUNBOOK.md` R1).
- **Open items** (`ISSUES_AND_FIXES.md` §B): no `app_id` for multi-tenancy; some Lambdas
  still read `os.environ` directly, not `secrets_loader`; signup grants only local
  Aarambh, not the spec'd multi-app free grant; universal Frappe backend for `/plans` is a
  separate, unverified repo. No automated tests — relies on `scripts/validate-*.cjs` +
  manual Chrome checks (blocked on browser MCP). Unmerged branches carry an in-flight
  DynamoDB→SQL migration (~218 call-sites/23 files) — `main` is verified.
