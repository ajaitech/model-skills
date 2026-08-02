# AivedhA Guard

## Goal

AiVedha Guard (npm `aivedha-guard`, live at `aivedha.ai`) is an AI-powered website security
audit platform. A user submits a URL; the backend runs a 41-step scan (SSL/TLS, DNS,
headers, XSS/SQLi/SSRF/XXE/JWT, AI risk synthesis), streams live progress, and produces a
PDF report, a score/grade, and — above a threshold — a public certificate + badge. Users:
site owners and security teams; GitHub Marketplace/Actions lets CI/CD trigger audits.
Monetized via plans (`aarambh` free → `chakra` enterprise) plus credit packs on Razorpay.
Part of the AiVibe SaaS suite, sharing Cognito SSO and one RDS.

## Core requirements

- Audit lifecycle is atomic: URL validate → credit check → scan → live progress (AppSync
  sub) → all items → PDF → certificate/badge if qualified → email → **credit deducted only
  after the email succeeds**. Findings shown to the owner unredacted, secrets included.
- Universal data (users/plans/subscriptions/credits/billing) belongs to the shared
  `aivibe_platform` RDS; app data lives in `aivedha_ai_*` tables.
- No hardcoded plan codes/prices/URLs/emails — config/env/DB sourced; `FALLBACK ONLY`
  fixtures labeled. Types match case-sensitively across `audit.types.ts` ↔ `schema.graphql`
  ↔ Lambda dicts. Secrets via `secrets_loader.py`. Razorpay is the default gateway; PayPal
  only if `VITE_ENABLE_PAYPAL=true` (`src/config/index.ts:208`).

## Tech stack

| Layer | Technology | Version | Source of truth |
|---|---|---|---|
| Frontend | React/TS, Vite+SWC, Tailwind/shadcn, react-router 6.30.3, react-query 5.90.21 | 18.3.1/5.9.3/7.3.1/3.4.19 | `package.json` |
| API primary | AWS AppSync GraphQL | id `sgig66576fhyvc2or2beq775ne` | `aws-appsync/schema.graphql:7` |
| API fallback | API Gateway + Lambda REST | `https://api.aivedha.ai/api` | `src/lib/api.ts:91` |
| Backend | AWS Lambda, Python (py3.12 target) | — | `deploy/deploy_lambda.sh:76` |
| Database | PostgreSQL, shared `aivibe_platform` RDS | — | `shared/db_connection.py` |
| Auth | Cognito Hosted UI, shared AiVibe pool | `us-east-1_S2Cpx3svp` | `src/lib/cognito.ts:3` |
| Hosting | S3 + CloudFront | — | `.github/workflows/deploy-production.yml` |
| Payments | Razorpay SDK (Python) | `razorpay==1.4.2` | `razorpay-handler/requirements.txt` |

Version `2.7.4`; no `aws-amplify` — hand-rolled native `WebSocket` on the `graphql-ws`
subprotocol (`src/lib/appsync-client.ts:268`).

## Build / run / deploy

- Prereqs: Node 22 (CI pins it), Python 3.12 + AWS CLI creds for Lambda work; copy
  `.env.example` → `.env`.
- `npm ci` → `npm run dev` (Vite on **port 8080**, `vite.config.ts:9`); `npm run lint`;
  `npm run build` (`prebuild` regenerates sitemap + RSS).
- Frontend deploys on push to `main`: lint → build → `s3 sync` → `index.html` no-cache
  re-upload → CloudFront invalidation → IndexNow ping.
- Lambda deploy is manual: `deploy/deploy_lambda.sh <short-name>` → function
  `aivedha-guardian-<short-name>` (`deploy_lambda.sh:35`); deps built
  `--platform manylinux2014_x86_64 --python-version 3.12`.
- No test suite. `scripts/validate-*.cjs` / `test-*.mjs` are contract checks, not tests.

## Architecture

Client-only Vite/React SPA on S3+CloudFront. Two backend paths coexist: AppSync GraphQL
fronting Lambda resolvers, with the `onAuditProgress` subscription for live progress; and a
legacy REST path via API Gateway (`src/lib/api.ts`) slated for retirement.
**29 Lambda function directories** under `aws-lambda/`, each with `lambda_function.py`
(`shared/`, `layers/`, `jwt-layer/` are not functions); scanner =
`security-crawler/security-audit-crawler.py`. DB access is direct psycopg2
(`shared/db_connection.py`); `shared/universal_db.py` also hits universal tables (see gaps).

## Naming conventions

- Components PascalCase one-per-file (`Dashboard.tsx`); other TS modules kebab-case;
  Lambda dirs kebab-case → `aivedha-guardian-<short-name>`.
- Tables prefixed `aivedha_ai_` (collision-proofing in the shared `public` schema), columns
  snake_case. Env: frontend `VITE_*`, backend upper-snake.
- GraphQL types PascalCase; fields dual-cased — camelCase canonical (`progressPercent`,
  `schema.graphql:198`) plus snake_case legacy aliases (`credit_used`).
- Plans `aarambh`,`raksha`,`suraksha`,`vajra`,`chakra` (`src/constants/apps.ts:59`).
  `app_id`=`'aivedha-guard'` (`shared/gateway_config.py:622`). Branches
  `<type>/<desc>-<YYYY-MM-DD>`.

## Data types & models — `database/aivedha_ai_schema.sql` (CREATE TABLE line)

| Table `aivedha_ai_…` | Key fields | Line |
|---|---|---|
| `audit_reports` | `report_id PK`,`user_id`,`status`,`security_score`,`grade`,`credit_used`,`scan_results:JSONB`,`items_total DEFAULT 41` (:103) | 35 |
| `audit_findings` | `finding_id PK`,`report_id FK`,`severity`,`evidence:TEXT` (unredacted),`cvss_score`,`ai_status` | 124 |
| `certificates` | `certificate_number PK`,`report_id FK`,`security_score`,`grade`,`revoked` | 169 |
| `scheduled_audits` | `schedule_id PK`,`cron_expr`,`active`,`next_run_at` | 195 |
| `addon_purchases` | `purchase_id PK`,`addon_code`,`provider` | 352 |

## API surface

`schema.graphql`: **30 Query, 82 Mutation, 1 Subscription** fields; client `appsync-api.ts`.

| Operation | Auth |
|---|---|
| `startAudit`/`getAuditStatus`/`startCicdAudit` | API key / per-user `X-API-Key` |
| `onAuditProgress` (sub) / `publishAuditProgress` | api_key+iam / iam-only |
| `authenticateUser`/`registerUser`/`googleAuth`/`authenticateGitHub` | API key |
| `createRazorpayOrder`/`Subscription`/`verifyPayment` | API key |
| `createApiKey`/`deleteApiKey`/`revokeApiKey` | `@aws_cognito_user_pools` |
| `handleGitHubWebhook` | HMAC (`GITHUB_WEBHOOK_SECRET`) |

## Security boundary

- CORS lives only in `aws-lambda/shared/cors.py` (REST Lambdas; AppSync does its own):
  allow-list `CORS_ALLOWED_ORIGINS`, default `aivedha.ai`/`www.`/`admin.` (:18);
  `resolve_origin()` echoes Origin only if allow-listed; `Allow-Credentials: true`,
  `GET,POST,PUT,DELETE,OPTIONS` (:43).
- Cognito Hosted UI at `auth.aivibe.cloud`. AppSync: `API_KEY` default + `AWS_IAM` extra
  provider; some fields `@aws_cognito_user_pools`.
- Secret NAMES only: `GITHUB_WEBHOOK_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, `PAYPAL_WEBHOOK_ID`,
  `RDS_HOST`, `RAZORPAY_KEY_ID`, `ADMIN_JWT_SECRET`; Secrets Manager ids `aivibe/*`.
- Public: marketing, `/pricing`, `/certificate/:num`, `/verify/:num`, blog, badges. Private
  (`<PrivateRoute>`): `/dashboard`, `/security-audit`, `/purchase`, `/profile`,
  `/scheduler`. `/admin/*` uses a separate admin-JWT path.

## Known gaps & risks

- **IaC gap**: the schema header claims "ALL operations served through AppSync," but
  `aws-appsync/cloudformation-template.yaml` declares only **2** `AWS::AppSync::Resolver`.
  Every other resolver exists only in the live API, outside checked-in IaC.
  `deploy/update_appsync_resolvers.py` does **not** create them — it lists live resolvers
  and rewrites their VTL templates (skipping `NoneDataSource`). Delete the API and the
  resolver set cannot be rebuilt from this repo.
- **Universal-data boundary broken by design**: target state is universal tables via
  `api.aivibe.cloud`; `shared/universal_db.py` uses direct SQL instead, since that API has
  no service-to-service auth path (its module header says so). Read/UPDATE only —
  CREATE/DROP forbidden. `shared/universal_api.py`, cited by
  `docs/important-ref-docs/MIGRATION_STATUS.md`, does not exist — stale docs.
- **README overstates coverage**: "21 modules, 178+ checks" (`README.md:30`, echoed in
  `src/constants/features.ts:123`) vs the crawler's fixed **41-item, 8-phase** pipeline
  (`security-audit-crawler.py:1672`; initialization, crawling, ssl_analysis, dns_analysis,
  header_analysis, vulnerability_detection, ai_analysis, report_generation). Keep the
  measured numbers. README also presents PayPal as the gateway; it is Razorpay.
- **TLS misconfig (P0)**: `api.aivedha.ai` serves the API Gateway default cert, not the
  `*.aivedha.ai` ACM cert — breaks direct API/CI-CD consumers
  (`docs/important-ref-docs/INFRA_RUNBOOK.md` R1). Raksha/Suraksha have empty Razorpay
  plan IDs (R3).
- **Open** (`docs/testing-needs/ISSUES_AND_FIXES.md`): no `app_id` column for
  multi-tenancy; some Lambdas read `os.environ` instead of `secrets_loader`; signup grants
  only local Aarambh, not the spec'd multi-app grant. Non-obvious failure: a Lambda
  deployed without `aivedha-guardian-common-layer` fails at import time.
