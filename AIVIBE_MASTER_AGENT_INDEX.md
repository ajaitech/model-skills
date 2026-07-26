<!--
  Copyright 2026 Aivibe, Aivedha, Ajairtech, and Aravind Jayamohan
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0
  Ownership Attribution: Aivibe & Founder Aravind Jayamohan. All rights reserved.
-->

# AiVibe Platform & Design Standards — Unified Master Agent Index

This document serves as the absolute, single source of truth and master index for all AI models (Claude Code, Codex, Gemini, Antigravity) operating across the AiVibe ecosystem. It consolidates all visual guidelines, architecture patterns, API contracts, and database structures.

---

## Part 1: The Master Visual & UX Standard (22 Strict Rules)

These 22 rules are **strictly mandatory** for all development, refactoring, and AI execution. No drafts, no mocks, and no exceptions.

### 1. Dynamic Font Color Contrast Inversion
Font color must always dynamically adapt to its background color. Use explicit contrast-derivation checking background luminance to guarantee readable WCAG AA/AAA compliance under any theme or background variation. Text must never blend with its background.

### 2. Rounded Edges Everywhere
Every single layout, frame, component, container, card, modal, or input field must have rounded corners (radius) added. Sharp 0-radius corners are strictly forbidden. Minimum base radius is 10px / `0.625rem`, scaling up by component size.

### 3. Mandatory 3D Hover Color Inversion on Clickables
Every clickable or action event component (menus, sub-menus, buttons, links, tabs, icons, toggles, etc.) must implement a hover state that dynamically inverts **all three** of the following colors simultaneously:
1. **Background Color** (Inverts)
2. **Font/Shape/Icon Color** (Inverts)
3. **Border Color** (Inverts)
Transition must be paired with a subtle, premium 3D lift/depth effect.

### 4. Compact Button Width (Never Width-Full)
Buttons across all components must keep a compact, content-aware width. Avoid long, verbose button labels. Specifically for buttons, setting `width: 100%` or `w-full` is strictly prohibited. Keep buttons elegant, compact, and centered or logically grouped.

### 5. Logo-Only Header, Fixed Footer & Floating Dock Menu
- **No Top Menu & No Header Bar**: The very top of the page should have no standard menu or headers, except for the logo positioned beautifully in the head.
- **One-Line Fixed Footer**: Always display a clean, fixed, one-line footer at the bottom containing links for: `Policy | Support | Terms & Conditions | All Rights Reserved`.
- **Floating Bottom View Docker**: Primary navigation must use a stylish, floating bottom docker-like menu (resembling a premium floating dock).
- **Floating Compact Side Menu**: If a side menu is necessary, it must be designed as a floating, dynamic, and compact sidebar panel.
- **Floating Login / Profile Orbit**: The login icon must float independently. Once authenticated, it must seamlessly morph with a smooth animation into a floating user profile menu complete with an animated "Log Off" icon.

### 6. Blurred-Background Popup Widget Login UI
Login must never occupy a separate standard page. It must always reside in an independent popup/modal widget overlay. The background page behind the popup must be dynamically blurred. The popup must support both existing users and first-time signups, providing integrated, elegant pre-signup forms.

### 7. Premium Apple & Google Sourced Visuals
Always benchmark and exceed the latest trending visuals, depth, layouts, and typography of official Apple and Google web designs. Produce pristine visual hierarchies, exceptional element visibility, and elegant high-contrast designs.

### 8. Zero Duplication & Intelligent Neural Layouts
Never duplicate headings, titles, labels, or functions across any screen or page. Use intelligent, neural layout thinking to strategically position each component on the UI/UX canvas, keeping layouts logical, spacious, and perfectly balanced.

### 9. 100% Smart Components & Click Minimization
Every component must be 100% smart, maximizing automated loaders or background fetching of values intelligently from APIs while keeping user-required clicks to the absolute minimum. No manual entry of data that can be retrieved automatically.

### 10. Maximum Stack Reusability
Reusability is a top priority. Variables, functions, utility hooks, components, classes, and file architectures must be designed with maximum, dry, and logical reusability in mind. Write once, reuse everywhere.

### 11. Zero Dead or Broken Code
Delete all duplicates, alternate files, unused variables, comments holding old code, or broken files immediately. Never retain dead, experimental, or commented-out logic in the project repository. Keep codebases lean and active.

### 12. Full Production-Ready Stage Only
No MVPs, mocks, stubs, placeholders, or hardcoded constants in the codebase. Every implementation must be production-ready and fully integrated to actual backend APIs, utilizing environment secrets.

### 13. Universal SSO, Multi-Tenant Isolation & Autoscale Plans
- **AWS Cognito Universal SSO**: Login must utilize the unified Cognito pool `us-east-1_S2Cpx3svp` at `auth.aivibe.cloud`.
- **Unified Platform Admin APIs**: User/org management, plans, support, credits, and tenant IDs must be managed using the global APIs at `api.aivibe.cloud`.
- **Tenant-Scoped DB & Row-Level Security (RLS)**: Database must be the universal RDS PostgreSQL database at `db.aivibe.cloud`. App-specific tables must use the app name as a prefix and contain a foreign-key relationship to user tables via `tenant_id`. Strict Row-Level Security (RLS) is a mandatory architectural standard.
- **Secrets Management**: AWS secrets must always be fetched from `secrets.aivibe.cloud`.
- **App Registrations**: Each new app must be registered in the central database with an `app_id` assigned to the default company entity (Aivibe & Aivedha).
- **Instant Signup Plans**: Every user signup must automatically be assigned to the free plan for all registered `app_id`s in the database, enabling immediate multi-platform trial access.

### 14. Poetic HTML Email Templates (SES Only)
All automatic alerts, notifications, and transactional emails must be configured with poetic, rich, attractive, and colorful HTML templates. Emails must use SES ready-to-use domains in AWS and must use standardized sender addresses (e.g., `no-reply@...` or `alerts@...`).

### 15. Transparent Pages & 3D Background Animations
UIs must feature a 3D animated background with transparent pages overlaid, giving a futuristic sense of depth and motion. When adding colors or accents, prioritize using a rich, vibrant shine of **Lime Green** or a rich, vibrant shine of **Pure Red**.

### 16. Stylish Top-Right Breadcrumbs
All pages must feature a stylish, clean breadcrumb navigation positioned on the top-right of the available empty space.

### 17. AWS-Style Precision Cards (Sharp Blades + Soft Shadow)
Visual borders of components and cards must have thin, sharp, blade-like outer outlines with extremely clean borders (resembling the cards on the official AWS website), blended with soft, elegant shadows that perfectly match the application's overall color theme. (Maintain the rounded corners specified in Rule 2 while ensuring card edges and thin outlines are precise and sharp-bladed).

### 18. Latest Dependency Versions Only
Always use the latest stable versions of all dependencies and packages. There is zero tolerance for skipping, delaying, or staying on older versions due to outstanding minor bugs or unfixed issues. Solve outstanding issues or adapt code, but keep packages fully up-to-date.

### 19. Complete Code-First (No Stories, Minimal Text)
Never rush to build or deploy. Complete 100% of the code architecture before attempting compilation or deployment. Keep all responses, explanations, and summaries to a single line. Avoid writing long narratives, documentation blocks, or stories. Focus strictly on clean, functional, running code.

### 20. Comprehensive Event Fault Tolerance (At least 2 Positive, 2 Negative)
Every single event, action, or page load function must feature bulletproof exception-handling. The implementation must explicitly handle and manage:
- At least **two** positive, fully successful paths.
- At least **two** negative, broken, interrupted, or offline scenario paths.
No component should crash or hang when a network or state failure occurs.

### 21. Case-Sensitive Naming Conventions
Follow case-sensitive, fully unified naming conventions across the frontend and backend. Always compile a naming-registry mapping list of variables, functions, and columns to eliminate odds or mismatches and keep the entire stack absolutely uniform.

### 22. 98%+ Grounding Confidence (No Hallucinations, Online Research)
Never judge prompts, assume facts, guess values, or hallucinate. Grounding confidence must stay at 98% and above for every single action. The model must actively search online for up-to-date information, libraries, APIs, and guidelines matching the specific context, and never deviate from the established codebase conventions unless executing a planned, 100% guaranteed improvement in both visual and functional aspects.

---

## Part 2: Platform API & Domain Specifications

### 1. Unified SaaS Platform: `api.aivibe.cloud`
The platform API handles core tenancy, plans, subscriptions, and wallet calculations.
- **Cognito SSO Authority**: User Pool `us-east-1_S2Cpx3svp` via issuer `https://auth.aivibe.cloud/us-east-1_S2Cpx3svp`. Verification JWKS resides at `https://auth.aivibe.cloud/us-east-1_S2Cpx3svp/.well-known/jwks.json`.
- **GraphQL Operations**:
  - `query GetActiveSubscription($userId: ID!)`: Resolves active subscription plan codes (`aarambh | raksha | suraksha | vajra | chakra`).
  - `query GetCreditWallet($userId: ID!)`: Fetches the current user balance, earned credits, and lifetime consumption ledger.
  - `mutation UseCredits($userId: ID!, $amount: Int!, $module: String!)`: Records a ledger debit entry for billing tracking.
- **Tenant Context Propagation**: Tenant ID is extracted strictly from the JWT `custom:tenant_id` claim at the API gateway layer. downstream services must never accept user-provided body params for tenancy.

### 2. AiVedha Guard: `aivedha.ai`
AI-powered security auditing and telemetry.
- **Auditing API**:
  - `POST /api/v1/audits/schedule`: Enqueues an audit request for a target domain.
  - `GET /api/v1/audits/reports/:id`: Returns structured CWE/CVSS/OWASP vulnerability findings and visual score grades.
  - `GET /api/v1/audits/progress`: Subscribes to SSE (Server-Sent Events) reporting real-time progress percentages.
- **Module Handlers**: Traces audit metrics through separate specialized sub-analyzers (Aura, Orbit, Seal).

### 3. VibeKaro: `vibekaro.ai`
Central routing, multi-currency checkout, and billing orchestration.
- **Billing API**:
  - `POST /api/v1/billing/checkout`: Initializes a PayPal or Razorpay transaction. Supports international cards and custom currency routing.
  - `POST /api/v1/billing/verify`: Webhook signature validator verifying Razorpay `X-Razorpay-Signature` or PayPal callbacks.
- **Gateway Rules**: Server-side pricing recalculation is mandatory. All transactions are scoped with unique client-side idempotency keys.

### 4. AiAmba: `aiamba.com` & `api.emmarkay.com`
IoT controller, factory orchestrator, and hardware telemetry gateway.
- **IoT API**:
  - `POST /api/v1/devices/register`: Registers edge computing devices and generates secure cryptographic access keys.
  - `POST /api/v1/telemetry/ingest`: High-throughput ingestion of hardware metrics, mapped to PostgreSQL partition boundaries.

### 5. Next.js Portal & Design Primitives: `aivedha.io`
Next.js React Portal architecture and unlayered design system primitives.
- **Design Tokens**: Token classes are imported via `@aivedha/ui/tokens.css`.
- **Utility Animation Classes (from `@import "./animations.css"`)**:
  - `av-anim-aurora`: looping aurora gradient hue drift (16s ping-pong)
  - `av-anim-gradient-text`: looping brand gradient background text-clip
  - `av-anim-shimmer-brand`: sky-violet-emerald sweep sheen
  - `av-anim-border-flow`: traveling border gradient working at any aspect ratio
- **D1/D2 Strict Compliance**: All local `<button>` or custom components inherit unlayered CSS rules forcing `border-radius >= 12px` and 3D hover-lifts.

### 6. Real-time Subscription Client: `aicippy.io`
ArjunA-powered multi-agent browser extension and WebSockets gateway.
- **Real-time Subscriptions**: Establishes WebSockets connection utilizing base64url encoded header authorization targeting AppSync WebSockets endpoints.

---

## Part 3: Universal Database Schema & Tenancy Structure (`db.aivibe.cloud`)

All applications share a single PostgreSQL database on AWS RDS (`db.aivibe.cloud`).

```
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
- **`aivedha.users`**: PK `user_id` (UUID), `cognito_sub` (VARCHAR), `email` (VARCHAR), `tenant_id` (UUID), `organization_id` (UUID), `role` (VARCHAR), `status` (VARCHAR), `profile` (JSONB).
- **`aivedha.organizations`**: PK `organization_id` (UUID), `slug` (VARCHAR), `owner_id` (UUID), `billing` (JSONB).
- **`aivedha.plans`**: PK `plan_id` (UUID), `plan_code` (VARCHAR), `credits_monthly` (INT), `features` (JSONB).
- **`aivedha.subscriptions`**: PK `subscription_id` (UUID), `user_id` (UUID), `plan_id` (UUID), `payment_provider` (VARCHAR), `external_subscription_id` (VARCHAR), `status` (VARCHAR), `period_start` (TIMESTAMP), `period_end` (TIMESTAMP).
- **`aivedha.credits`**: PK `user_id` (UUID), `balance` (INT), `lifetime_earned` (INT), `lifetime_used` (INT).
- **`aivedha.credit_transactions`**: PK `transaction_id` (UUID), `user_id` (UUID), `delta` (INT), `balance_before` (INT), `balance_after` (INT), `transaction_type` (VARCHAR), `reference_id` (UUID).
- **`aivedha.payment_transactions`**: PK `transaction_id` (UUID), `provider` (VARCHAR), `external_transaction_id` (VARCHAR), `amount` (DECIMAL), `status` (VARCHAR), `idempotency_key` (VARCHAR).

### 2. Guard Schema (`public.aivedha_ai_*` & `guard_*`)
- **`aivedha_ai_audit_reports`**: PK `report_id` (UUID), `user_id` (UUID), `url` (VARCHAR), `status` (VARCHAR), `score` (DECIMAL), `grade` (VARCHAR), `vulnerabilities` (JSONB).
- **`guard_payments`**: PK `payment_id` (UUID), `invoice_number` (VARCHAR), `amount` (DECIMAL), `gateway` (VARCHAR), `gateway_event_id` (VARCHAR).

### 3. VibeKaro Schema (`public.vibekaro_ai_*`)
- **`vibekaro_ai_projects`**: PK `project_id` (UUID), `tenant_id` (UUID), `name` (VARCHAR).
- **`vibekaro_ai_workspace_sessions`**: PK `session_id` (UUID), `project_id` (UUID), `tenant_id` (UUID).

### 4. Row-Level Security Policies (`public.aivedha_net_*`)
- Row-Level Security (RLS) is strictly enforced via GUC config setting:
  `SET app.tenant_id = '<verified tenant>'`
- RLS Policy Rule:
  ```sql
  CREATE POLICY tenant_isolation ON public.aivedha_net_tool_usage
  FOR ALL TO public
  USING (tenant_id = NULLIF(current_setting('app.tenant_id', true), '')::uuid);
  ```

---

## Part 4: Case-Sensitive Unified Naming Standard & Datatypes

| Field / Variable Name | Context | PostgreSQL Datatype | TS / Dart Datatype | Naming Convention |
|---|---|---|---|---|
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
