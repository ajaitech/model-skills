# AiAmbA Factory Studio

## Goal

AiFactory (by AiAmbA/AiAbmA) is a multi-tenant SaaS console walking a factory owner from a static-IP router connection through device discovery, protocol/stream routing, AI-operator assignment, and governed operation — no cloud/network/AI skill required. Audience: industrial SMB/mid-market owners (`docs/requirements/4-marketing-raw-data/`), pilot site → multi-site fleets (edge GPU, cameras, robotics, PLC/CNC).

## Core requirements

- Every route in `frontend/console/src/navigation/aiabmaMenu.ts` resolves to a readiness-gated screen or documented recovery action (`noDeadScreenRecoveryAction`) — never a dead screen.
- Tenant isolation: forced Postgres RLS on `app.tenant_key` for every `aifactory_*` table (`005_aifactory_rls.sql`); frontend never exposes raw tokens.
- Device commands need an approval boundary plus a salted per-device consent passcode (iterations: see Data types); secrets are always `secret_ref` pointers, never plaintext.
- `npm run ci` (policy/UI-wiring/API-contract audits, typecheck, lint, tests, build, sensitive-content scan, npm audit) gates every deploy. Dev loop: `npm run dev|build`; deploys via `npm run deploy:{database,api-gateway,appsync,frontend}`.

## Tech stack

| Layer | Technology | Version | Source of truth |
|---|---|---|---|
| Frontend | React / Vite / TypeScript / @siemens/ix(-react) | 19.2.8 / 8.2.0 / 7.0.2 / 5.1.1 | `frontend/console/package.json` |
| Runtime/PM | Node.js / npm workspaces | >=26.0.0 / npm@11.12.1 | root `package.json` |
| Backend | FastAPI / Uvicorn / Mangum / Pydantic / boto3 / psycopg / cryptography | 0.139.0 / 0.51.0 / 0.20.0 / 2.12.5 / 1.43.18 / 3.3.4 / 46.0.7 | `middle-layer/api/pyproject.toml` |
| Database / Python | PostgreSQL+RLS (pgvector unused) / >=3.11 source, Lambda `python3.13` | unpinned | `database/schema/*.sql`, deploy script |
| AWS / edge / CI | Cognito JWT, API Gateway v2, AppSync (unused) / Caddy 2 on EC2 / GitHub Actions | — | `dev/scripts/deploy-*.mjs`, `infra/aws/runtime/` |

Regions split: deploys default `ap-south-1`; Cognito pool `us-east-1` — cross-region by design.

## Architecture

`frontend/console` (React SPA) calls a REST/JWT API at `VITE_AIABMA_API_BASE` (`.env.production` actually sets the legacy `VITE_AIAMBA_API_BASE` twin; code accepts both → `https://api.emmarkay.com`). The API is one first-party FastAPI app (`middle-layer/api/app/main.py`, 7,379 lines), Mangum-wrapped, one Lambda behind API Gateway v2; Postgres (RLS by `tenant_key`) is system of record. Frontend ships via S3 + SSM RunCommand to an EC2 host running Caddy for 5 frontend domains.

Workspace packages: `@aiabma/console` (`frontend/console`) — live product console + API client (`src/api/{backend,progress,tenant,ip}.ts`); `@aiabma/protocol-catalog` (`libs/protocol-catalog`) — adapter/device-class/module registry → Capability Map; `aiabma-api` (`middle-layer/api/app`, Python) — production FastAPI app + Lambda handler.

## Tooling inventory

**Real count: 59** rows in `integrationModuleRows` (`libs/protocol-catalog/src/index.ts`), the registry behind the Capability Map — closest to the "50+" claim. Cross-check: `upstreams.tsv`+`packages.tsv` = 55+1 rows. **4/59 rows reference service dirs absent on disk.** All 55 real referenced dirs carry a marker reading "sanitized reference import — wrapper/service pending" (2, `ui-flowise-chat`/`ui-langflow-designer`, also "dependency-closed curated feature subset"). Catalog `runtimeStatus` (29 adapter-ready/23 needs-wrapper/7 reference-only) is self-reported and rosier — trust the markers.

- Dirs: frontend 11, shared-apps 15, plugins 4, services 14, libs 6, middle-layer 5, database 5 (= 60).
- First-party only: `frontend/console`, `libs/protocol-catalog`, `middle-layer/api/app` (inside the FastAPI clone dir), `services/discovery-runner` (spec only: `adapter-contracts.yml`), `database/seed-masters`+`seed-test-data`, and `database/schema/00*.sql` (inside the postgres clone dir).
- Catalog rows with no dir: `services/{fanuc-focas-bridge,secure-eye-bridge,endpoint-agent,pymcprotocol-bridge}`.

## Naming conventions

Spelling is **inconsistent in-repo**: npm scope/folder use `@aiabma/*`/`AiAbmA_UI`; 54 markers are `AIAMBA-MODULE.md` (`AiAmbA-*` ids), 1 (`database/schema`) is `AIABMA-MODULE.md`. AWS resources still use `aiamba` (`aiamba-gpu-worker-asg`); scripts fall back across both (`AIABMA_COGNITO_CLIENT_ID || AIAMBA_COGNITO_CLIENT_ID`). DB tables snake_case prefixed `aifactory_`; request bodies snake_case remapped from camelCase (`staticIp`→`static_ip`); route ids kebab-case.

## Data types & models

All Postgres, RLS-forced except last; first four from `003_aifactory_core.sql`.

| Entity | Fields (name : type) |
|---|---|
| aifactory_sites | id:uuid, tenant_key:text, static_ip:inet, router_secret_ref:text, status:text |
| aifactory_device_registry | id:uuid, site_id:uuid, ip_address:inet, mac_address:macaddr, device_class:text, protocol_hints:text[], confidence:numeric(5,2) |
| aifactory_protocol_channels / _device_consent_keys | adapter_id:text, operating_mode:text; passcode_hash/salt:text, hash_iterations:int (default 210000, ≥120000) |
| aifactory_agents / _agent_skills | agent_code:text, role_name:text, provider_profile_id:uuid; skill_code:text, training_state:text |
| aifactory_runtime.runtime_auth_challenges (`002_runtime_state.sql`, no RLS) | transaction_hash:text (sha256, 64-char PK), cognito_state_ciphertext:text (KMS), expires_at:timestamptz |

## API surface

Base `https://api.emmarkay.com`. Handlers in `main.py`; client in `frontend/console/src/api/`. DTOs `Backend*`/`BackendListResponse<T>`; JWT = Cognito bearer + tenant header. No DELETE routes exist.

| Operation | Method / Path |
|---|---|
| Health / auth (public; /auth/session is JWT) | GET /health; POST /auth/{login,challenge,refresh,logout,password/forgot,password/confirm}; GET /auth/session |
| Catalogs (public) | GET /catalog/{router-profiles,protocol-adapters,ai-providers}, /network/isp-profile, /templates/email |
| GPU / discovery / onboarding / routes | GET /gpu/status, POST /gpu/{requests,extend}; POST /discovery/jobs, /onboarding/static-ip/nvidia-agent-runs(+/{id}/cancel), /routes |
| Tenant / provider profiles | GET/PUT /tenant/profile; POST /tenant/onboarding; GET /provider-profiles, PUT /provider-profiles/{id} |
| Sites AND duplicate factories | POST/GET /sites, GET/PUT /sites/{id} — same CRUD again at /factories(/{id}) |
| Factory subresources | GET/POST\|PUT /factories/{site}/{devices,channels,streams,cameras,router,edge-server,discovery-runs}; POST …devices/{code}/{edge-policy,agent-commands}, PUT …/consent-passcode |
| Progress + governance lists | POST/GET /progress/sessions(+/{id}, +/events); GET/POST /agents (+PUT /agents/{code}, +/{code}/skills), /agent-skills, /orchestration-sessions, /design-assets, /simulation-runs, /training-runs, /notifications; GET /audit/events (legacy /audit-events) |
| WebSocket (defined, unreachable in prod) | WS /ws/progress/{id}, /ws/sites/{id}, /ws/factories/{site}/devices/{code}/agents/{agent} |

## CORS & headers

API Gateway CORS comes from `AIABMA_CORS_ORIGINS`/`AIAMBA_CORS_ORIGINS`, else the 5 production frontend domains (`deploy-api-gateway-lambda.mjs`); no CORS middleware in `main.py`. `infra/aws/runtime/Caddyfile` adds strict CSP (`connect-src 'self' https://api.emmarkay.com wss://api.emmarkay.com https://auth.aivibe.cloud`), HSTS, frame-deny, nosniff, and reverse-proxies `/api/*` for same-origin.

## Security boundary

Cognito hosted UI + PKCE (`auth.aivibe.cloud`, pool `us-east-1_S2Cpx3svp`) plus username/password via FastAPI `/auth/*`; JWT authorizer `AiVibeCognitoJwt` gates all but 12 public API routes (+OPTIONS). Secret NAMES only, never values: `AIABMA_DATABASE_SECRET_ID`, `AIABMA_DATABASE_HOST`, `AIABMA_COGNITO_CLIENT_ID`, `AIFACTORY_NGC_SECRET_ID`, `AIFACTORY_AUTH_CHALLENGE_KMS_ALIAS`, `DATABASE_URL` (legacy `AIAMBA_*` twins).

## Known gaps & risks

- Realtime unwired: `main.py` defines 3 WS routes and the frontend opens `/ws/progress/{id}` (`src/api/progress.ts`), but API Gateway is HTTP-only (no WS upgrade) and the deployed AppSync API (`deploy-appsync-realtime.mjs`) has no frontend caller. `/sites/*` vs `/factories/*` duplication unresolved.
- Catalog drift: 4/59 capability-map rows point at non-existent service dirs; self-reported `runtimeStatus` rosier than markers (see Tooling).
- `aiamba`/`aiabma` drift only partly bridged by dual env-var fallbacks; `pnpm-workspace.yaml` is vestigial — repo is npm-managed (`package-lock.json`, `packageManager: npm@11.12.1`).
- `middle-layer/{public-api,server,async-api,nim-proxy}`, `plugins/`, `shared-apps/`, most `services/` and `frontend/ui-*` are sanitized clones, not running code — check the module marker before calling anything "deployed".
