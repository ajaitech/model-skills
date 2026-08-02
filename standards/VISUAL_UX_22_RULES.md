## Applies when

Building, changing, or reviewing any user-facing UI in any AiVibe product.

---

# The Master Visual & UX Standard (22 Strict Rules)

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
