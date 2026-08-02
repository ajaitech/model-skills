# AiAmbA Factory Studio

## Goal

AiFactory (built by AiAmbA/AiAbmA) is a multi-tenant SaaS console that walks a factory owner from a static-IP router connection through device discovery, protocol/stream routing, AI-operator assignment, and governed operation — without cloud/network/AI engineering skill. Audience: industrial SMB/mid-market owners (`docs/requirements/4-marketing-raw-data/`), staged from one pilot site to multi-site fleets with edge GPU, cameras, robotics, PLC/CNC control.

## Core requirements

- Every route in `aiabmaMenu.ts` resolves to a readiness-gated screen or documented recovery action (`noDeadScreenRecoveryAction`) — never a dead screen.
- Tenant isolation is forced Postgres RLS on `app.tenant_key` for every `aifactory_*` table (`005_aifactory_rls.sql`); Cognito JWT authorizer guards protected routes; frontend never exposes raw tokens.
- Device commands require an approval boundary plus a salted per-device consent passcode (≥120,000 hash iterations); secrets are always `secret_ref` pointers, never plaintext.
- `npm run ci` (policy/UI-wiring/API-contract audits, typecheck, lint, test, build, sensitive-content scan) gates every deploy.

## Tech stack

| Layer | Technology | Version | Source of truth |
|---|---|---|---|
| Frontend | React / Vite / TypeScript / @siemens/ix(-react) | 19.2.8 / 8.2.0 / 7.0.2 / 5.1.1 | `frontend/console/package.json` |
| Runtime/PM | Node.js / npm workspaces | >=26.0.0 / npm@11.12.1 | `package.json`, `.nvmrc` |
| Backend | FastAPI / Uvicorn / Mangum / Pydantic / boto3 / psycopg / cryptography | 0.139.0 / 0.51.0 / 0.20.0 / 2.12.5 / 1.43.18 / 3.3.4 / 46.0.7 | `middle-layer/api/pyproject.toml` |
| Database / Python | PostgreSQL+RLS (pgvector unused) / >=3.11 source, Lambda `python3.13` | unpinned | `database/schema/*.sql` |
| AWS / edge / CI | Cognito JWT, API Gateway v2, AppSync (unused) / Caddy 2 on EC2 / GitHub Actions | pool `us-east-1_S2Cpx3svp` | `dev/scripts/deploy-*.mjs`, `infra/aws/runtime/install.sh` |

## Architecture

`frontend/console` (React SPA) calls a REST/JWT API at `VITE_AIABMA_API_BASE` (prod `https://api.emmarkay.com`, same-origin via Caddy) and Cognito's hosted domain for OAuth2/PKCE. The API is one first-party FastAPI app (`middle-layer/api/app/main.py`, ~7,400 lines), Mangum-wrapped, one Lambda behind API Gateway with a Cognito JWT authorizer; Postgres (RLS by `tenant_key`) is system of record. AppSync is deployed for realtime but unused by the client. Frontend ships via S3 + SSM RunCommand to an EC2 host running Caddy across 4 domains. Non-first-party dirs (see Tooling inventory) are **sanitized upstream OSS reference imports**, not production services.

| Workspace package | Purpose | Path |
|---|---|---|
| `@aiabma/console` | Live product: console, routing, API client | `frontend/console` |
| `@aiabma/protocol-catalog` | Adapter/device-class/integration-module registry, feeds Capability Map | `libs/protocol-catalog` |
| `aiabma-api` (Python) | Production FastAPI app + Lambda handler | `middle-layer/api/app` |

## Tooling inventory

**Real count: 59** — from `integrationModuleRows` in `libs/protocol-catalog/src/index.ts`, the registry driving the Capability Map screen (`source-capability-map`) — closest to the "50+" claim. Method: counted the array, cross-checked vs. filesystem (`find frontend middle-layer services plugins shared-apps libs database -mindepth 1 -maxdepth 1 -type d` → 60 dirs) and `upstreams.tsv`+`packages.tsv` (55+1=56 rows). **4/59 entries reference paths absent on disk** (metadata only). Of 55 real dirs, all carry a module marker file; 53 say `wrapper/service pending`, 2 say `curated feature subset`. Catalog `runtimeStatus` is rosier (29 `adapter-ready`, 23 `needs-wrapper`, 7 `reference-only`) than the markers or the live-API review (~13 route groups wired end-to-end).

| Layer | Dirs | First-party | Missing (catalog-only) |
|---|---|---|---|
| frontend/shared-apps/plugins | 11/15/4 | console/—/— | — |
| services | 14 | discovery-runner | 4: fanuc-focas-bridge, secure-eye-bridge, endpoint-agent, pymcprotocol-bridge |
| libs/middle-layer/database | 6/5/5 | protocol-catalog/api/seed-masters+seed-test-data | — |

## Naming conventions

Spelling is **inconsistent in-repo**: npm scope/folder use `@aiabma/*`/`AiAbmA_UI` (54 module markers say `AIAMBA-MODULE.md`/`AiAmbA-*`; 1, `database/schema`, says `AIABMA-MODULE.md`/`AiAbmA-postgres`). AWS resources still use `aiamba` (`aiamba-gpu-worker-asg`); scripts fall back across both (`AIABMA_COGNITO_CLIENT_ID || AIAMBA_COGNITO_CLIENT_ID`). DB tables: snake_case, prefixed `aifactory_`. Frontend DTOs: PascalCase `Backend*`; bodies snake_case, remapped from camelCase (`staticIp`→`static_ip`). Route ids: kebab-case.

## Data types & models

| Entity | Fields (name : type) | Store | Defined in |
|---|---|---|---|
| aifactory_sites | id:uuid, tenant_key:text, static_ip:inet, router_secret_ref:text, status:text | Postgres RLS | `database/schema/003_aifactory_core.sql` |
| aifactory_device_registry | id:uuid, site_id:uuid, ip_address:inet, device_class:text, protocol_hints:text[], confidence:numeric | Postgres RLS | same |
| protocol_channels / device_consent_keys | adapter_id:text, operating_mode:text; passcode_hash/salt:text, hash_iterations:int(≥120000) | Postgres RLS | same |
| agents / agent_skills | agent_code:text, role_name:text, provider_profile_id:uuid; skill_code:text, training_state:text | Postgres RLS | same |
| runtime_auth_challenges | transaction_hash:text (sha256 PK), cognito_state_ciphertext:text (KMS), expires_at:timestamptz | Postgres | `002_runtime_state.sql` |

## API surface

Base `https://api.emmarkay.com`. Defined in `frontend/console/src/api/{backend.ts,progress.ts}` + `middle-layer/api/app/main.py`. Auth: none or JWT (Cognito bearer + tenant header). Responses: typed DTOs prefixed `Backend*`/`BackendListResponse<T>`.

| Operation | Method / Path | Request | Auth |
|---|---|---|---|
| Health | GET /health | — | none |
| Login/challenge/pw-reset/session/refresh/logout | POST /auth/{login,challenge,password/forgot,password/confirm,refresh,logout}, GET /auth/session | user_id, password, challenge_token | none/JWT |
| ISP profile / catalogs | GET /network/isp-profile, GET /catalog/{router-profiles,protocol-adapters,ai-providers} | query static_ip | none (catalogs unused) |
| Tenant / sites / provider profiles | GET/PUT /tenant/profile, POST /tenant/onboarding; GET /sites, PUT/GET /sites/{id}; GET/PUT /provider-profiles(+/{id}) | static_ip, router_*, model_id, secret_ref | JWT |
| Discovery, devices(+edge-policy/consent-passcode/agent-commands), channels, streams | POST /discovery/jobs; POST/GET /factories/{site}/{devices,channels,streams}+subpaths | site_id, device_code, adapter_id, passcode | JWT |
| Progress sessions/events (+ socket gap) | POST /progress/sessions(+/events), GET /progress/sessions(+/{id}); WS /ws/progress/{id} not deployed | kind, step_id, status | JWT |
| Audit/agents/skills/orchestration/design/simulation/training/notifications | GET /audit/events, /agents, /agent-skills, /orchestration-sessions, /design-assets, /simulation-runs, /training-runs, /notifications | — | JWT |

## CORS & headers

`deploy-api-gateway-lambda.mjs` sets API Gateway CORS from `AIABMA_CORS_ORIGINS`/`AIAMBA_CORS_ORIGINS`, else defaults to the 5 production domains. `infra/aws/runtime/Caddyfile` adds a strict CSP (`connect-src 'self' https://api.emmarkay.com wss://api.emmarkay.com https://auth.aivibe.cloud`), HSTS, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, and reverse-proxies `/api/*` for same-origin traffic. No CORS middleware in `main.py` itself — API-Gateway-managed only.

## Security boundary

Cognito hosted UI + PKCE (`auth.aivibe.cloud`, pool `us-east-1_S2Cpx3svp`) plus username/password via FastAPI `/auth/*`; JWT authorizer `AiVibeCognitoJwt` gates protected routes; Postgres RLS on `app.tenant_key` is forced. Secret NAMES only, never values: `AIABMA_DATABASE_SECRET_ID`, `AIABMA_DATABASE_HOST`, `AIABMA_COGNITO_CLIENT_ID`, `AIFACTORY_NGC_SECRET_ID`, `AIFACTORY_AUTH_CHALLENGE_KMS_ALIAS`, `DATABASE_URL`, `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` (legacy `AIAMBA_*` too). Public: marketing domains + 12 API routes; else JWT + tenant header required.

## Known gaps & risks

- Realtime unwired: frontend calls `/ws/progress/*`/`/ws/sites/*` but API Gateway is HTTP-only; deployed AppSync sits unused. `/sites/*` vs `/factories/*` route duplication also unresolved.
- Catalog drift: 4/59 capability-map entries point at non-existent service dirs; self-reported `runtimeStatus` is rosier than per-module markers and the live-route wiring review.
- `aiamba`/`aiabma` naming drift only partly bridged by dual env-var fallbacks; `pnpm-workspace.yaml` is vestigial (repo is npm-managed).
- Most of `middle-layer/{public-api,server,async-api,nim-proxy}`, `plugins/`, `shared-apps/`, most `services/`/`frontend/ui-*` are sanitized clones, not running code — check module markers before calling something "deployed."
