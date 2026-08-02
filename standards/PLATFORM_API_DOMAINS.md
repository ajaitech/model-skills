## Applies when

Integrating with, or changing, any AiVibe platform API or domain.

---

# Platform API & Domain Specifications

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
