## Applies when

Naming any identifier, column, or file, or choosing a datatype, anywhere in the platform.

---

# Case-Sensitive Unified Naming Standard & Datatypes

| Field / Variable Name | Context | PostgreSQL Datatype | TS / Dart Datatype | Naming Convention |
| --- | --- | --- | --- | --- |
| `tenant_id` | Database / API | `UUID` | `string` | snake_case |
| `organization_id` | Database / API | `UUID` | `string` | snake_case |
| `user_id` | Database / API | `UUID` | `string` | snake_case |
| `plan_code` | Subscriptions | `VARCHAR(50)` | `string` | snake_case |
| `credits_monthly` | Plans | `INT` | `number` | snake_case |
| `idempotency_key` | Payments | `VARCHAR(255)` | `string` | snake_case |
| `delta` | Credits | `INT` | `number` | snake_case |
| `balance_before` | Credit Transactions | `INT` | `number` | snake_case |
| `balance_after` | Credit Transactions | `INT` | `number` | snake_case |
| `external_transaction_id` | Payments | `VARCHAR(255)` | `string` | snake_case |
| `invoice_number` | Invoices | `VARCHAR(50)` | `string` | snake_case |
| `current_period_start` | API Response | `TIMESTAMP` | `Date` / `string` | snake_case |
| `current_period_end` | API Response | `TIMESTAMP` | `Date` / `string` | snake_case |
| `percent_used` | API Response | `DECIMAL(5,2)` | `number` | snake_case |
| `currentStage` | Telemetry Response | `VARCHAR(100)` | `string` | camelCase |
| `progressPercent` | Telemetry Response | `DECIMAL(5,2)` | `number` | camelCase |
