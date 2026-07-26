---
name: aivibe-platform
description: Map of the AiVibe ecosystem — the real repos, products, production domains, and the shared platform (auth.aivibe.cloud Cognito SSO, api.aivibe.cloud GraphQL, db.aivibe.cloud RDS PostgreSQL) that ties them together. Use whenever a task spans these projects — aicippy (aicippy.io/.com), aivedha.ai (AiVedha Guard), aivedha.com (www + magic ERP monorepo), vibekaro.ai (payments), Poolmate/VibeMyCar — or asks how they authenticate, bill, or share data. Pairs with aivibe-sso-cognito and aivibe-db-schema.
---

# AiVibe ecosystem map (grounded in the repos)

One family of SaaS apps (AiVibe Software Services) sharing **one** auth, **one**
platform API, and **one** RDS database. Each app owns its own app-prefixed tables;
the shared/universal tables (users, plans, subscriptions, credits) are owned only
by `api.aivibe.cloud`.

## Repos → products → domains
| Repo (`/Users/aj/Dev-Apps/…`) | Product | Domains | Stack / deploy |
|---|---|---|---|
| `aicippy-cli/aicippy-cli` | **AiCippy** — multi-agent CLI + Chrome extension | `aicippy.io`, `aicippy.com`, gateway `api.aicippy.io` (ArjunA) | Python 3.14, Bedrock; PyPI via GitHub Actions |
| `aivedha-guard` | **AiVedha Guard** — AI security-audit SaaS | `aivedha.ai`, `api.aivedha.ai` | React/Vite + Lambda(Python) + RDS; S3+CloudFront |
| `aivedha-platform` (monorepo) | **AiVedha** — `apps/www` (marketing) + `apps/magic` (Cognitive ERP) | `aivedha.com`, `www.aivedha.com`, `magic.aivedha.com` | React + Express + Socket.IO; **GCP Cloud Run** |
| `vibekaro-ai` | **VibeKaro** — payments + engagement, central payment router | `vibekaro.ai` (+ `/pricing`, `/payment/*`) | React/Vite |
| `Poolmate_V3.3_Source_Code` | **VibeMyCar (Poolmate)** — ride-sharing | (package `com.vibemycar.aivibe`) | Flutter; minimal/legacy tree locally |

> Naming: **aivedha.ai = Guard** (the audit repo). **aivedha.com = the monorepo**
> (www + magic ERP). aicippy.io/.com = the CLI product. vibekaro.ai = payments hub.
> The central **`aivibe_apps`** RDS table is the authoritative app registry
> (mirrored offline in `aivedha-guard/src/constants/apps.ts`).

## The shared platform (the glue)
- **Auth — `auth.aivibe.cloud`** (Cognito pool `us-east-1_S2Cpx3svp`, us-east-1): single
  sign-on for all apps; per-app client ids; `custom:tenant_id` claim carries tenancy.
  Details + flows → **aivibe-sso-cognito**.
- **API — `api.aivibe.cloud/graphql`**: universal platform API for user/tenant, plans,
  subscriptions, credits/wallet, secrets, analytics. All apps read billing/identity here.
- **DB — `db.aivibe.cloud`** (RDS PostgreSQL): shared `aivedha` schema (universal) +
  per-app `*_ai_*` / `guard_*` / `aivedha_net_*` tables. Full structure → **aivibe-db-schema**.
- **Billing**: plans `aarambh / raksha / suraksha / vajra / chakra`; payments via
  PayPal + Razorpay; pricing surfaced at `vibekaro.ai/pricing`.

## Hard rules (from the repos — respect them)
- **DynamoDB is forbidden** in AiVedha Guard (PostgreSQL-on-RDS only).
- A repo **MUST NOT** create/alter another app's tables; universal tables are
  deployed only by `api.aivibe.cloud`.
- **VibeMyCar/Poolmate**: do NOT rebrand from "Poolmate" without explicit confirmation.
- Never hardcode secrets — Secrets Manager / env only.

## When working across apps
1. Identify the app + its repo from the table above.
2. Auth always flows through `auth.aivibe.cloud` (aivibe-sso-cognito).
3. Identity/billing data lives in the shared `aivedha` schema via `api.aivibe.cloud`
   (aivibe-db-schema) — don't query another app's tables directly.
4. For AWS work use the aws-cli / aws-bedrock skills; for tenant isolation use
   bedrock-multitenant-security / aws-multitenant-saas.
