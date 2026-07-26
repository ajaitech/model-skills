---
name: aivibe-db-schema
description: The real database structure of db.aivibe.cloud (PostgreSQL on AWS RDS, us-east-1) shared by all AiVibe apps — the universal `aivedha` schema (users, organizations, plans, subscriptions, credits, payments, addons, coupons, referrals), AiVedha Guard's `aivedha_ai_*` audit tables, `guard_*` payment tables, VibeKaro `vibekaro_ai_*` workspace tables, and `aivedha_net_*` (with RLS). Use for any schema/data/query/migration task on the platform DB — which table holds what, ownership boundaries, keys, and tenant isolation. Grounded in aivedha-guard/database/*.sql; inferred items are flagged.
---

# db.aivibe.cloud schema (PostgreSQL/RDS, us-east-1)

Shared DB. **Ownership is strict:** universal `aivedha.*` tables are deployed ONLY by
`api.aivibe.cloud`; each app owns its prefixed tables and **must not** create/alter
another app's. (DynamoDB is forbidden for Guard.) Authoritative DDL:
`/Users/aj/Dev-Apps/aivedha-guard/database/aivedha_ai_schema.sql` (+ `payment_tables.sql`).
Read that file for exact column types before writing migrations.

## Universal / shared — schema `aivedha` (owned by api.aivibe.cloud)
The identity + billing core every app reads (via GraphQL, not direct cross-app SQL):
- **users** (user_id PK, cognito_sub, email, **tenant_id**, organization_id, role, status, profile JSONB…) — universal identity across all login methods.
- **organizations** (organization_id PK, slug, owner_id→users, billing…).
- **plans** (plan_id PK, plan_code [`aarambh|raksha|suraksha|vajra|chakra`], price_*, credits_monthly, features JSONB).
- **subscriptions** (subscription_id PK, user_id, plan_id/plan_code, payment_provider, external_subscription_id, status [active|cancelled|expired|past_due|trialing], period_*).
- **credits** (user_id unique, balance ≥0, lifetime_earned/used) + **credit_transactions** (signed credits, balance_before/after, transaction_type, reference_*).
- **payment_transactions** (provider, external_transaction_id, amount, status, idempotency_key).
- **addons** + **user_addons** (credit packs `credits-5..100`, `scheduled_audits`, `whitelabel_cert`).
- **coupons**, **referrals**. Views: `v_user_dashboard`, `v_active_coupons`.

These map to the aicippy GraphQL ops: `activeSubscription`→subscriptions+plans,
`creditWallet`→credits, `useCredits`→credit_transactions+credits.

## AiVedha Guard — `public.aivedha_ai_*` (13 tables, owned by aivedha-guard)
audit pipeline: **aivedha_ai_audit_reports** (report_id PK, user_id, url, status,
score/grade, *_count, big JSONB result blocks) → **aivedha_ai_audit_findings**
(finding_id, report_id, severity, cwe/cvss/owasp) → **aivedha_ai_audit_module_status**
(per-module progress). Plus **aivedha_ai_certificates**, **aivedha_ai_scheduled_audits**
(cron), **aivedha_ai_audit_emails** (SES log), **aivedha_ai_webhook_idempotency**,
**aivedha_ai_blogs / _blog_comments / _blog_ratings**, **aivedha_ai_support_tickets**,
**aivedha_ai_websocket_connections**, **aivedha_ai_addon_purchases**.
Tenancy: `user_id` (soft FK to `aivedha.users`); identity enforced via JWT at the API.

## Guard payments — schema `aivedha`, `guard_*` (owned by aivedha-guard)
**guard_payments**, **guard_subscriptions**, **guard_invoices** (number `AG-INV-YYYYMM-#####`,
GST fields), **guard_payment_events** (webhook audit, unique [gateway,gateway_event_id]),
**guard_gateway_config**. Dual gateway PayPal + Razorpay. Helpers:
`aivedha.generate_invoice_number()`, `aivedha.get_active_subscription(user)`,
`aivedha.record_payment_event(...)`.

## VibeKaro — `public.vibekaro_ai_*` (⚠ inferred from migration files, full DDL not in repo)
**vibekaro_ai_projects**, **_workspace_sessions**, **_builds**, **_agent_memory**
(per-agent arjuna/aikutty session memory, TTL via `ttl_epoch` + reaper Lambda),
**_ws_connections**, **_conversations** (+ `_phase_signoffs`, referenced, DDL not found).
Tenancy: explicit **tenant_id** + project_id on every table.

## AiVedha.net — `public.aivedha_net_*` (RLS-enforced)
**aivedha_net_donations**, **_tool_usage**, **_user_tool_access_history**,
**_visit_counters** (public), **_cookie_consent**. **Row-Level Security**:
`tenant_isolation` policy via `current_setting('app.tenant_id')` — set
`SET app.tenant_id = '<verified tenant>'` per request.

## AiCippy — no app tables
The CLI has **no** RDS tables of its own; it uses the universal `aivedha.*` data via
`api.aivibe.cloud` GraphQL. (`tenantSecret`/`trackAnalyticsEvent` reference
platform entities whose DDL wasn't found in the repos.)

## Multi-tenancy summary
| Layer | Mechanism |
|---|---|
| `aivedha.*` (universal) | tenant_id/organization_id columns; isolation enforced at the **API** (api.aivibe.cloud), not DB RLS |
| `aivedha_ai_*` / `guard_*` | user_id (→ aivedha.users); JWT identity at API |
| `vibekaro_ai_*` | explicit tenant_id + project_id |
| `aivedha_net_*` | **DB RLS** via `app.tenant_id` GUC |

## Discipline
- Read the actual `.sql` for exact column types/constraints before any migration.
- Never create/alter a table outside its owning app; never query another app's tables
  directly — go through `api.aivibe.cloud`.
- Items marked ⚠ inferred/not-found: confirm against the live DB or the owning repo
  before relying on them. Don't invent columns.
