## Applies when

Reading, writing, or migrating any table in the shared platform database.

---

# Universal Database Schema & Tenancy Structure (`db.aivibe.cloud`)

All applications share a single PostgreSQL database on AWS RDS (`db.aivibe.cloud`).

```text
+---------------------------------------------------------------------------------+
|                                 aivedha schema                                  |
|                                                                                 |
|  +--------------------+      +--------------------+      +--------------------+ |
|  |       users        |      |   organizations    |      |       plans        | |
|  | ------------------ |      | ------------------ |      | ------------------ | |
|  | user_id (PK)       |o----+| organization_id(PK)|      | plan_id (PK)       | |
|  | email              |      | owner_id           |      | plan_code          | |
|  | tenant_id          |      +--------------------+      | credits_monthly    | |
|  | organization_id    |                                  +--------------------+ |
|  +--------------------+                                             ^           |
|            |                                                        |           |
|            |                                                        |           |
|            |                 +--------------------+                 |           |
|            |                 |   subscriptions    |                 |           |
|            |                 | ------------------ |                 |           |
|            +---------------->| subscription_id(PK)|o----------------+           |
|            |                 | plan_id            |                             |
|            |                 +--------------------+                             |
|            v                                                                    |
|  +--------------------+      +--------------------+      +--------------------+ |
|  |      credits       |      |credit_transactions |      |payment_transactions| |
|  | ------------------ |      | ------------------ |      | ------------------ | |
|  | user_id (PK)       |o----+| transaction_id (PK)|      | transaction_id (PK)| |
|  | balance            |      | user_id            |      | amount             | |
|  +--------------------+      +--------------------+      | idempotency_key    | |
|                                                          +--------------------+ |
+---------------------------------------------------------------------------------+
```

### 1. Core Schema (`aivedha`)

- **`aivedha.users`** — PK `user_id` (UUID), `cognito_sub` (VARCHAR), `email` (VARCHAR), `tenant_id` (UUID), `organization_id` (UUID), `role` (VARCHAR), `status` (VARCHAR), `profile` (JSONB).
- **`aivedha.organizations`** — PK `organization_id` (UUID), `slug` (VARCHAR), `owner_id` (UUID), `billing` (JSONB).
- **`aivedha.plans`** — PK `plan_id` (UUID), `plan_code` (VARCHAR), `credits_monthly` (INT), `features` (JSONB).
- **`aivedha.subscriptions`** — PK `subscription_id` (UUID), `user_id` (UUID), `plan_id` (UUID), `payment_provider` (VARCHAR), `external_subscription_id` (VARCHAR), `status` (VARCHAR), `period_start` (TIMESTAMP), `period_end` (TIMESTAMP).
- **`aivedha.credits`** — PK `user_id` (UUID), `balance` (INT), `lifetime_earned` (INT), `lifetime_used` (INT).
- **`aivedha.credit_transactions`** — PK `transaction_id` (UUID), `user_id` (UUID), `delta` (INT), `balance_before` (INT), `balance_after` (INT), `transaction_type` (VARCHAR), `reference_id` (UUID).
- **`aivedha.payment_transactions`** — PK `transaction_id` (UUID), `provider` (VARCHAR), `external_transaction_id` (VARCHAR), `amount` (DECIMAL), `status` (VARCHAR), `idempotency_key` (VARCHAR).

### 2. Guard Schema (`public.aivedha_ai_*` & `guard_*`)

- **`aivedha_ai_audit_reports`** — PK `report_id` (UUID), `user_id` (UUID), `url` (VARCHAR), `status` (VARCHAR), `score` (DECIMAL), `grade` (VARCHAR), `vulnerabilities` (JSONB).
- **`guard_payments`** — PK `payment_id` (UUID), `invoice_number` (VARCHAR), `amount` (DECIMAL), `gateway` (VARCHAR), `gateway_event_id` (VARCHAR).

### 3. VibeKaro Schema (`public.vibekaro_ai_*`)

- **`vibekaro_ai_projects`** — PK `project_id` (UUID), `tenant_id` (UUID), `name` (VARCHAR).
- **`vibekaro_ai_workspace_sessions`** — PK `session_id` (UUID), `project_id` (UUID), `tenant_id` (UUID).

### 4. Row-Level Security Policies (`public.aivedha_net_*`)

Row-Level Security is strictly enforced via the GUC configuration setting:

```sql
SET app.tenant_id = '<verified tenant>';
```

RLS policy rule:

```sql
CREATE POLICY tenant_isolation ON public.aivedha_net_tool_usage
FOR ALL TO public
USING (tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::uuid);
```
