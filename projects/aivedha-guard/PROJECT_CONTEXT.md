# AivedhA Guard

## Goal

Website security-audit platform at `aivedha.ai`. Submit a URL → fixed **41-item / 8-phase** scan →
live AppSync progress → PDF + 0–10 score + grade; certificate and public badge only at score
**≥ 7.0 with zero critical/high findings** (`shared/certificate_policy.py:17,133`). Plans
`aarambh`→`chakra` + credit packs on Razorpay; the RDS and Cognito pool are shared with the rest
of the AiVibe suite. `aivedha-guard` **2.7.4**, private.

## Core requirements

- The audit lifecycle is atomic (`docs/important-ref-docs/EXECUTION_MANDATE.md` §3) and ends
  PDF → certificate/badge if qualified → email → **credit deducted only after the email succeeds**
  (`security-audit-crawler.py:12115`, idempotency key `audit-<report_id>`). No finding is
  redacted, hidden or summarised.
- No hardcoded plan codes/prices/URLs/emails. Secrets via `secrets_loader.py` (Secrets Manager
  `aivibe/*` → env fallback). Razorpay is the default gateway; PayPal only if
  `VITE_ENABLE_PAYPAL=true` (`src/config/index.ts:208`).

## Tech stack

| Layer | Technology | Source |
|---|---|---|
| Frontend | React 18.3.1, TS 5.9.3, Vite 7.3.1+SWC, Tailwind 3.4.19 + shadcn, react-router 6.30.3, react-query 5.90.21 | `package.json` |
| API primary | AppSync GraphQL, api id `sgig66576fhyvc2or2beq775ne`, Lambda resolvers via the `shared/appsync_handler.py` shim | `schema.graphql:7` |
| API legacy | API Gateway REST `https://api.aivedha.ai/api`, retiring | `src/lib/api.ts:91` |
| Backend | Lambda, Python 3.12 target, **29 entrypoints** | `deploy_lambda.sh:76` |
| Database | PostgreSQL `aivibe_platform`, search_path `public` | `shared/db_connection.py:62,68` |
| Auth | Cognito Hosted UI `auth.aivibe.cloud`, pool `us-east-1_S2Cpx3svp` | `src/lib/cognito.ts:3` |
| Payments | `razorpay==1.4.2` + `setuptools==70.3.0` (py3.12 dropped `pkg_resources`) | `razorpay-handler/requirements.txt` |

The 29 entrypoints are 27 `lambda_function.py` + `security-crawler/security-audit-crawler.py` +
`js-renderer/handler.py`; `shared/`, `layers/`, `jwt-layer/` are not functions. No `aws-amplify`
— hand-rolled `WebSocket` on `graphql-ws` (`appsync-client.ts:268`). No AWS CDK.

## Build / run / deploy

- Prereqs: Node 22 (`deploy-production.yml:59`); Lambdas need Python 3.12 + `pip` + `zip` + AWS
  CLI creds; `js-renderer` also Docker + ECR. `.env.example` → `.env` (7 required `VITE_*` keys).
- `npm ci` → `npm run dev` (Vite on **port 8080**, `vite.config.ts:9`); `npm run lint`;
  `npm run build` (`prebuild` regenerates sitemap + RSS).
- Push to `main` deploys the frontend (lint → build → s3 sync → `index.html` no-cache re-upload
  → CloudFront invalidation → IndexNow); a **`grep -RInE "localhost|127.0.0.1|staging" dist`
  guard fails the build on any match**. `deploy/deploy_frontend.sh` is the manual equivalent.
- Lambda (zip): `deploy/deploy_lambda.sh <dir> [--smoke]` → `aivedha-guardian-<dir>` (`:35`).
  Stages `shared/*.py` at zip root, then installs a per-Lambda `requirements.txt` (only 5 dirs
  have one) with AWS's cross-platform wheel flags `--platform manylinux2014_x86_64
  --implementation cp --python-version 3.12 --only-binary=:all:` (`:76`, per
  `docs.aws.amazon.com/lambda/latest/dg/python-package.html`).
- Lambda (container): `js-renderer` is an **image** function built by
  `js-renderer/build_deploy.sh` (Playwright/Chromium → ECR); its entry is `handler.py`, so
  `deploy_lambda.sh` rejects it by design (`:44`).
- AppSync: `deploy/update_appsync_resolvers.py` **updates existing** resolvers' VTL only — it
  never creates them.
- No test runner in `package.json`; two `unittest` modules sit under `shared/` and
  `scripts/validate-*.cjs`/`test-*.mjs` are hand-run contract checks CI never runs.

## Data model & conventions

`database/aivedha_ai_schema.sql` — 13 `aivedha_ai_*` tables in the shared `public` schema, columns
snake_case. `audit_reports` (:35) holds `security_score`, `grade`, `credit_used`,
`scan_results JSONB`, `items_total DEFAULT 41` (:103); `audit_findings` (:124) is one row per
occurrence with `evidence` never redacted. `migrations/2026_findings_ai_queue.sql` adds
`finding_signature`/`ai_status`/`ai_severity` for asynchronous Bedrock enrichment over SQS.
`app_id`=`'aivedha-guard'` (`shared/gateway_config.py:622`).

## API surface

`schema.graphql`: **30 Query, 56 Mutation, 1 Subscription** = 87 root fields (parsed from each root
type block); documents in `src/lib/appsync-api.ts`. Fields are dual-cased: camelCase
canonical (`progressPercent` :198) + snake_case legacy aliases (`credit_used` :195).
`API_KEY` is the default auth provider; exceptions: `onAuditProgress` (`@aws_api_key`+`@aws_iam`,
:1257), `publishAuditProgress` (`@aws_iam`, :1153), `listApiKeys`/`createApiKey`/`deleteApiKey`/
`revokeApiKey` (`@aws_cognito_user_pools`, the only four), `handleGitHubWebhook` (HMAC), and the
CI/CD path, which authenticates a per-tenant `X-API-Key` (`crawler:11028`).

## Security boundary

- CORS is **not** centralised: 14 of 29 functions import `shared/cors.py` (env allow-list
  `CORS_ALLOWED_ORIGINS`, default apex/`www.`/`admin.` `:18`, `Allow-Credentials: true` `:43`).
  The rest hand-roll headers, several emitting `Access-Control-Allow-Origin: '*'`
  (`badge-generator:55`, `secure-badge:68`, `crawler:11021`) — editing `cors.py` misses those.
- `/admin/*` is a separate auth path: `adminToken` in localStorage verified by `admin-auth` with
  `ADMIN_JWT_SECRET`, not Cognito. `<PrivateRoute>` (`src/App.tsx:100`) redirects anonymous users
  to `/` with a login popup, not a 403; `/certificate/:n` and `/verify/:n` are public.
- Secret NAMES only: `GITHUB_WEBHOOK_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, `PAYPAL_WEBHOOK_ID`,
  `ADMIN_JWT_SECRET`, `BADGE_TOKEN_SECRET`, `RDS_HOST`, `RDS_SECRET_ARN`.

## Known gaps & risks

- **IaC gap.** `schema.graphql:11` says all operations go through AppSync, yet
  `cloudformation-template.yaml` declares **1 NONE data source + 2 NONE-backed resolvers** and no
  Lambda data source — the 87 fields' real resolvers exist only in the live API. It also won't
  deploy: `DefinitionS3Location: ./schema.graphql` is a local path where CloudFormation
  needs an S3 location (`AWS::AppSync::GraphQLSchema` ref at
  `docs.aws.amazon.com/AWSCloudFormation/`), and `PlaceholderQueryResolver` binds
  `Query._placeholder`, absent from the schema.
- **Universal-data boundary broken by design.** `db_connection.py:8-11` bans universal-table
  queries here; `universal_db.py` imports its `execute_query`/`execute_update` anyway and runs
  direct SQL on 10 `public.*` universal tables because `api.aivibe.cloud` offers no
  service-to-service auth (`:6-15`). Read/UPDATE only. `shared/universal_api.py` (~10 cites in
  `MIGRATION_STATUS.md`) and `shared/invoice_generator.py` (`ISSUES_AND_FIXES.md` E9) do not exist
  — stale docs.
- **Coverage overstated.** "21 modules, 178+ checks" (`README.md:30`, `Hero.tsx:1430`,
  `FAQ.tsx:914`; "All 21 modules execute", `EXECUTION_MANDATE.md` §3.5) vs 41 items in 8 phases
  (`security-audit-crawler.py:1673`: initialization 4, crawling 5, ssl 5, dns 5, header 4,
  vulnerability 10, ai 4, report 4). Use the measured numbers; README also still names PayPal.
- **TLS misconfig (P0, open).** `api.aivedha.ai` serves the API Gateway default
  `*.execute-api.us-east-1.amazonaws.com` cert (`INFRA_RUNBOOK.md` R1; `ISSUES_AND_FIXES.md` C1
  🟥), breaking direct REST consumers and the Vite dev proxy (`vite.config.ts:12-14`,
  `secure: true`). The GitHub Action is unaffected — it targets `audit.aivedha.ai`.
- **Two Lambda deploy scripts.** `scripts/deploy-lambdas.sh` covers 8 functions, gates on a
  hardcoded account ID and never installs `requirements.txt`; using it for `razorpay-handler`
  ships without the SDK → `ImportModuleError` (`deploy_lambda.sh:66-72`). Use
  `deploy/deploy_lambda.sh`.
- **Layer contract unreproducible.** `deploy_lambda.sh:18-20` says
  `aivedha-guardian-common-layer` ships psycopg2 + boto3 + jwt + requests; checked-in
  `aws-lambda/layers/python/` holds 10 packages, none of them psycopg2 or boto3, and PyJWT lives
  in `jwt-layer/`. A function deployed without the layer fails at import.
- **Open ledger** (`docs/testing-needs/ISSUES_AND_FIXES.md`): B4 no `app_id` column; B5 some
  Lambdas read `os.environ` not `secrets_loader`; B6 signup grants only local Aarambh; C2–C6
  field drift `audit.types.ts` ↔ Lambda responses. R3 Raksha/Suraksha Razorpay plan IDs are empty
  (`src/config/index.ts:102-105`), so those tiers cannot check out.
