<!--
  Copyright 2026 Aivibe, Aivedha, Ajairtech, and Aravind Jayamohan
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0
  Ownership Attribution: Aivibe & Founder Aravind Jayamohan. All rights reserved.
-->

# AiVibe Platform & Design Standards — Unified Master Agent Index

This document is the absolute, single source of truth and master index for all AI models (Claude Code, Codex, Gemini, Antigravity) operating across the AiVibe ecosystem. It consolidates all visual guidelines, architecture patterns, API contracts, and database structures.

**These rules are mandatory in every project, without exception.** They apply to every repository, every application, every branch, and every task — new builds, refactors, bug fixes, reviews, and deployments alike. No drafts, no mocks, no partial adoption. An agent that cannot read this document must stop and report the exact failure rather than proceed from memory.

Canonical URL:

```text
https://raw.githubusercontent.com/ajaitech/model-skills/main/AIVIBE_MASTER_AGENT_INDEX.md
```

---

## Part 1: The Master Visual & UX Standard (22 Strict Rules)

These 22 rules are **strictly mandatory** for all development, refactoring, and AI execution. No drafts, no mocks, and no exceptions.

### 1. Dynamic Font Color Contrast Inversion

Font color must always dynamically adapt to its background color. Use explicit contrast derivation that checks background luminance to guarantee readable WCAG AA/AAA compliance under any theme or background variation. Text must never blend with its background.

### 2. Rounded Edges Everywhere

Every single layout, frame, component, container, card, modal, or input field must have rounded corners. Sharp `0`-radius corners are strictly forbidden. Minimum base radius is `10px` / `0.625rem`, scaling up by component size.

### 3. Mandatory 3D Hover Color Inversion on Clickables

Every clickable or action event component (menus, sub-menus, buttons, links, tabs, icons, toggles, and similar) must implement a hover state that dynamically inverts **all three** of the following colors simultaneously:

1. **Background color** — inverts.
2. **Font, shape, and icon color** — inverts.
3. **Border color** — inverts.

The transition must be paired with a subtle, premium 3D lift and depth effect.

### 4. Compact Button Width (Never Width-Full)

Buttons across all components must keep a compact, content-aware width. Avoid long, verbose button labels. Specifically for buttons, setting `width: 100%` or `w-full` is strictly prohibited. Keep buttons elegant, compact, and either centered or logically grouped.

### 5. Logo-Only Header, Fixed Footer & Floating Dock Menu

- **No top menu and no header bar** — the very top of the page carries no standard menu or header, except the logo positioned beautifully in the head.
- **One-line fixed footer** — always display a clean, fixed, one-line footer at the bottom containing: `Policy | Support | Terms & Conditions | All Rights Reserved`.
- **Floating bottom view docker** — primary navigation must use a stylish, floating bottom docker-like menu resembling a premium floating dock.
- **Floating compact side menu** — if a side menu is necessary, it must be a floating, dynamic, compact sidebar panel.
- **Floating login / profile orbit** — the login icon must float independently. Once authenticated it must seamlessly morph, with a smooth animation, into a floating user profile menu complete with an animated "Log Off" icon.

### 6. Blurred-Background Popup Widget Login UI

Login must never occupy a separate standard page. It must always reside in an independent popup or modal widget overlay. The background page behind the popup must be dynamically blurred. The popup must support both existing users and first-time signups, providing integrated, elegant pre-signup forms.

### 7. Premium Apple & Google Sourced Visuals — Liquid Glass (iOS 27)

Always benchmark and exceed the latest trending visuals, depth, layouts, and typography of official Apple and Google web designs. Produce pristine visual hierarchies, exceptional element visibility, and elegant high-contrast designs.

The mandatory material system is **Liquid Glass, as used in iOS 27**. Every surface — dock, sidebar, modal, popup, card, toolbar, sheet, and overlay — must render as a translucent glass layer:

- **Translucency and blur** — use real backdrop blur (`backdrop-filter: blur(...) saturate(...)`), never a flat opaque fill. Content behind the surface must remain perceptible.
- **Specular edge** — a bright `1px` inner highlight along the top and leading edges, with a darker trailing edge, so the surface reads as a lensed physical layer rather than a tinted rectangle.
- **Adaptive tint** — the glass tint samples the underlying background and adapts with the active theme. A fixed grey tint is forbidden.
- **Depth stacking** — overlapping glass layers must differ in blur radius and opacity so the hierarchy stays readable.
- **Motion and refraction** — glass responds to hover, scroll, and drag with subtle refraction and parallax, paired with the 3D lift required by Rule 3.
- **Legibility guard** — Rule 1's contrast inversion is computed against the **effective composited background** behind the glass, not against the glass tint alone.

### 8. Zero Duplication & Intelligent Neural Layouts

Never duplicate headings, titles, labels, or functions across any screen or page. Use intelligent, neural layout thinking to position each component strategically on the UI/UX canvas, keeping layouts logical, spacious, and perfectly balanced.

### 9. 100% Smart Components & Click Minimization

Every component must be fully smart, maximizing automated loaders and background fetching of values from APIs while keeping user-required clicks to the absolute minimum. No manual entry of data that can be retrieved automatically.

### 10. Maximum Stack Reusability

Reusability is a top priority. Variables, functions, utility hooks, components, classes, and file architectures must be designed with maximum, DRY, logical reusability. Write once, reuse everywhere.

### 11. Zero Dead or Broken Code

Delete all duplicates, alternate files, unused variables, comments holding old code, and broken files immediately. Never retain dead, experimental, or commented-out logic in the project repository. Keep codebases lean and active.

### 12. Full Production-Ready Stage Only

No MVPs, mocks, stubs, placeholders, or hardcoded constants in the codebase. Every implementation must be production-ready and fully integrated with actual backend APIs, using environment secrets.

### 13. Universal SSO, Multi-Tenant Isolation & Autoscale Plans

- **AWS Cognito universal SSO** — login must use the unified Cognito pool `us-east-1_S2Cpx3svp` at `auth.aivibe.cloud`.
- **Firebase Auth exception for pre-existing Firebase projects** — applications already built on Firebase, for example **VibeMyCar**, may continue to use **Firebase Authentication social login** (Google, Apple, Facebook, phone/OTP) instead of the Cognito Hosted UI. This exception applies only where the Firebase project predates AiVibe SSO adoption; it is never available to new applications, which must use Cognito. The Firebase identity must still be mapped to an `aivedha.users` row carrying `tenant_id`, `organization_id`, and `app_id`, so tenancy, plans, credits, billing, and Row-Level Security behave identically to Cognito-backed applications. The Firebase ID token must be verified server-side against the Firebase JWKS before any tenant context is derived.
- **Unified platform admin APIs** — user and organization management, plans, support, credits, and tenant IDs must be managed through the global APIs at `api.aivibe.cloud`.
- **Tenant-scoped database & Row-Level Security** — the database must be the universal RDS PostgreSQL instance at `db.aivibe.cloud`. App-specific tables must use the app name as a prefix and carry a foreign-key relationship to the user tables via `tenant_id`. Strict Row-Level Security is a mandatory architectural standard.
- **Secrets management** — AWS secrets must always be fetched from `secrets.aivibe.cloud`.
- **App registrations** — each new app must be registered in the central database with an `app_id` assigned to the default company entity (Aivibe & Aivedha).
- **Instant signup plans** — every user signup must automatically be assigned the free plan for all registered `app_id` values, enabling immediate multi-platform trial access.

### 14. Poetic HTML Email Templates (SES Only)

All automatic alerts, notifications, and transactional emails must use poetic, rich, attractive, colorful HTML templates. Emails must be sent through SES-ready domains in AWS using standardized sender addresses, such as `no-reply@...` or `alerts@...`.

### 15. Transparent Pages & 3D Background Animations

UIs must feature a 3D animated background with transparent pages overlaid, giving a futuristic sense of depth and motion. The overlaid surfaces follow the Liquid Glass material system defined in Rule 7. When adding colors or accents, prioritize a rich, vibrant shine of **Lime Green** or a rich, vibrant shine of **Pure Red**.

### 16. Stylish Top-Right Breadcrumbs

All pages must feature stylish, clean breadcrumb navigation positioned at the top-right of the available empty space.

### 17. AWS-Style Precision Cards (Sharp Blades + Soft Shadow)

Visual borders of components and cards must have thin, sharp, blade-like outer outlines with extremely clean borders, resembling the cards on the official AWS website, blended with soft, elegant shadows that match the application's overall color theme. Maintain the rounded corners specified in Rule 2 and the specular glass edge specified in Rule 7 while keeping card edges and thin outlines precise and sharp-bladed.

### 18. Latest Dependency Versions Only

Always use the latest stable versions of all dependencies and packages. There is zero tolerance for skipping, delaying, or staying on older versions because of outstanding minor bugs or unfixed issues. Solve outstanding issues or adapt the code, but keep packages fully up to date.

### 19. Complete Code-First (No Stories, Minimal Text)

Never rush to build or deploy. Complete 100% of the code architecture before attempting compilation or deployment. Keep all responses, explanations, and summaries to a single line. Avoid long narratives, documentation blocks, or stories. Focus strictly on clean, functional, running code.

### 20. Comprehensive Event Fault Tolerance (At Least 2 Positive, 2 Negative)

Every single event, action, or page-load function must feature bulletproof exception handling. The implementation must explicitly handle:

- At least **two** positive, fully successful paths.
- At least **two** negative, broken, interrupted, or offline scenario paths.

No component may crash or hang when a network or state failure occurs.

### 21. Case-Sensitive Naming Conventions

Follow case-sensitive, fully unified naming conventions across the frontend and backend. Always compile a naming-registry mapping list of variables, functions, and columns to eliminate mismatches and keep the entire stack absolutely uniform.

### 22. 98%+ Grounding Confidence (No Hallucinations, Online Research)

Never judge prompts, assume facts, guess values, or hallucinate. Grounding confidence must stay at 98% or above for every single action. The model must actively search online for up-to-date information, libraries, APIs, and guidelines matching the specific context, and never deviate from established codebase conventions unless executing a planned, fully guaranteed improvement in both visual and functional aspects.

---

## Part 2: Platform API & Domain Specifications

### 1. Unified SaaS Platform: `api.aivibe.cloud`

The platform API handles core tenancy, plans, subscriptions, and wallet calculations.

- **Cognito SSO authority** — user pool `us-east-1_S2Cpx3svp`. Login and OAuth2 use the Hosted UI custom domain `https://auth.aivibe.cloud`. Token verification uses the pool's own OIDC endpoints, never the custom domain: issuer `https://cognito-idp.us-east-1.amazonaws.com/us-east-1_S2Cpx3svp`, JWKS `https://cognito-idp.us-east-1.amazonaws.com/us-east-1_S2Cpx3svp/.well-known/jwks.json`. Both values are confirmed by the pool's discovery document at `.well-known/openid-configuration`.
- **GraphQL operations**:
  - `query GetActiveSubscription($userId: ID!)` — resolves active subscription plan codes (`aarambh | raksha | suraksha | vajra | chakra`).
  - `query GetCreditWallet($userId: ID!)` — fetches the current user balance, earned credits, and lifetime consumption ledger.
  - `mutation UseCredits($userId: ID!, $amount: Int!, $module: String!)` — records a ledger debit entry for billing tracking.
- **Tenant context propagation** — tenant ID is extracted strictly from the JWT `custom:tenant_id` claim at the API gateway layer. Downstream services must never accept user-provided body parameters for tenancy.

### 2. AiVedha Guard: `aivedha.ai`

AI-powered security auditing and telemetry.

- **Auditing API**:
  - `POST /api/v1/audits/schedule` — enqueues an audit request for a target domain.
  - `GET /api/v1/audits/reports/:id` — returns structured CWE/CVSS/OWASP vulnerability findings and visual score grades.
  - `GET /api/v1/audits/progress` — subscribes to Server-Sent Events reporting real-time progress percentages.
- **Module handlers** — traces audit metrics through separate specialized sub-analyzers (Aura, Orbit, Seal).

### 3. VibeKaro: `vibekaro.ai`

Central routing, multi-currency checkout, and billing orchestration.

- **Billing API**:
  - `POST /api/v1/billing/checkout` — initializes a PayPal or Razorpay transaction. Supports international cards and custom currency routing.
  - `POST /api/v1/billing/verify` — webhook signature validator verifying the Razorpay `X-Razorpay-Signature` header or PayPal callbacks.
- **Gateway rules** — server-side pricing recalculation is mandatory. All transactions are scoped with unique client-side idempotency keys.

### 4. AiAmba: `aiamba.com` & `api.emmarkay.com`

IoT controller, factory orchestrator, and hardware telemetry gateway.

- **IoT API**:
  - `POST /api/v1/devices/register` — registers edge computing devices and generates secure cryptographic access keys.
  - `POST /api/v1/telemetry/ingest` — high-throughput ingestion of hardware metrics, mapped to PostgreSQL partition boundaries.

### 5. Next.js Portal & Design Primitives: `aivedha.io`

Next.js React portal architecture and unlayered design system primitives.

- **Design tokens** — token classes are imported via `@aivedha/ui/tokens.css`.
- **Utility animation classes** (from `@import "./animations.css"`):
  - `av-anim-aurora` — looping aurora gradient hue drift, 16s ping-pong.
  - `av-anim-gradient-text` — looping brand gradient background text-clip.
  - `av-anim-shimmer-brand` — sky, violet, and emerald sweep sheen.
  - `av-anim-border-flow` — traveling border gradient working at any aspect ratio.
- **D1/D2 strict compliance** — all local `<button>` elements and custom components inherit unlayered CSS rules forcing `border-radius >= 12px` and 3D hover lifts.

### 6. Real-Time Subscription Client: `aicippy.io`

ArjunA-powered multi-agent browser extension and WebSockets gateway.

- **Real-time subscriptions** — establishes a WebSockets connection using base64url-encoded header authorization targeting AppSync WebSockets endpoints.

---

## Part 3: Universal Database Schema & Tenancy Structure (`db.aivibe.cloud`)

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

---

## Part 4: Case-Sensitive Unified Naming Standard & Datatypes

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
