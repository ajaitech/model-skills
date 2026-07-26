---
name: world-class-design
description: >-
  Strict UI/UX Master design standard. Contains 22 mandatory global rules covering dynamic inverting contrast, rounded components with razor-sharp thin borders, compact action components (never width-full buttons), custom layouts (docker menu, transparent 3D animation page backgrounds, top-right breadcrumbs), popup logins with SSO, multi-tenant RDS, email SES configurations, zero dead/commented code, 2-positive/2-negative load scenarios, and unified case-sensitive stack naming conventions with >= 98% grounding confidence.
when_to_use: >-
  Always load BEFORE building, styling, refining, or reviewing any UI screen, codebase, API, database structure, or system event.
---

# Master World-Class Design — The Strict 22-Point Standard

This is the definitive global design standard. These 22 rules are **strictly mandatory** for all development, refactoring, and AI execution. There are no exceptions, no development/draft modes, and no compromises.

---

## 1. Dynamic Font Color Contrast Inversion
Font color must always dynamically adapt to its background color. Use explicit, mathematical, or utility-based contrast-derivation (such as checking background luminance/lightness or utilizing auto-contrast features) to guarantee readable WCAG AA/AAA compliance under any theme or background variation. Text must never blend with its background.

## 2. Rounded Edges Everywhere
Every single layout, frame, component, container, card, modal, or input field must have rounded corners (radius) added. Sharp 0-radius corners are strictly forbidden. The exact boundary radius must scale proportionally based on component size, but a minimum base radius must be enforced at the base layer.

## 3. Mandatory 3D Hover Color Inversion on Clickables
Every clickable or action event component (menus, sub-menus, buttons, links, tabs, icons, toggles, etc.) must implement a hover state that dynamically inverts **all three** of the following colors simultaneously:
1. **Background Color** (Inverts)
2. **Font/Shape/Icon Color** (Inverts)
3. **Border Color** (Inverts)
This transition must be paired with a subtle, premium 3D lift/depth effect.

## 4. Compact Button Width (Never Width-Full)
Buttons across all components must keep a compact, content-aware width. Avoid long, verbose button labels. Specifically for buttons, setting `width: 100%` or `w-full` is strictly prohibited. Keep buttons elegant, compact, and centered or logically grouped.

## 5. Logo-Only Header, Fixed Footer & Floating Dock Menu
- **No Top Menu & No Header Bar**: The very top of the page should have no standard menu or headers, except for the logo positioned beautifully in the head.
- **One-Line Fixed Footer**: Always display a clean, fixed, one-line footer at the bottom containing links for: `Policy | Support | Terms & Conditions | All Rights Reserved`.
- **Floating Bottom View Docker**: Primary navigation must use a stylish, floating bottom docker-like menu (resembling a premium floating dock).
- **Floating Compact Side Menu**: If a side menu is necessary, it must be designed as a floating, dynamic, and compact sidebar panel.
- **Floating Login / Profile Orbit**: The login icon must float independently. Once authenticated, it must seamlessly morph with a smooth animation into a floating user profile menu complete with an animated "Log Off" icon.

## 6. Blurred-Background Popup Widget Login UI
Login must never occupy a separate standard page. It must always reside in an independent popup/modal widget overlay. The background page behind the popup must be dynamically blurred. The popup must support both existing users and first-time signups, providing integrated, elegant pre-signup forms.

## 7. Premium Apple & Google Sourced Visuals
Always benchmark and exceed the latest trending visuals, depth, layouts, and typography of official Apple and Google web designs. Produce pristine visual hierarchies, exceptional element visibility, and elegant high-contrast designs.

## 8. Zero Duplication & Intelligent Neural Layouts
Never duplicate headings, titles, labels, or functions across any screen or page. Use intelligent, neural layout thinking to strategically position each component on the UI/UX canvas, keeping layouts logical, spacious, and perfectly balanced.

## 9. 100% Smart Components & Click Minimization
Every component must be 100% smart, maximizing automated loaders or background fetching of values intelligently from APIs while keeping user-required clicks to the absolute minimum. No manual entry of data that can be retrieved automatically.

## 10. Maximum Stack Reusability
Reusability is a top priority. Variables, functions, utility hooks, components, classes, and file architectures must be designed with maximum, dry, and logical reusability in mind. Write once, reuse everywhere.

## 11. Zero Dead or Broken Code
Delete all duplicates, alternate files, unused variables, comments holding old code, or broken files immediately. Never retain dead, experimental, or commented-out logic in the project repository. Keep codebases lean and active.

## 12. Full Production-Ready Stage Only
No MVPs, mocks, stubs, placeholders, or hardcoded constants in the codebase. Every implementation must be production-ready and fully integrated to actual backend APIs, utilizing environment secrets.

## 13. Universal SSO, Multi-Tenant Isolation & Autoscale Plans
- **AWS Cognito Universal SSO**: Login must utilize the unified Cognito pool at `auth.aivibe.cloud`.
- **Unified Platform Admin APIs**: User/org management, plans, support, credits, and tenant IDs must be managed using the global APIs at `api.aivibe.cloud`.
- **Tenant-Scoped DB & Row-Level Security (RLS)**: Database must be the universal RDS PostgreSQL database at `db.aivibe.cloud`. App-specific tables must use the app name as a prefix and contain a foreign-key relationship to user tables via `tenant_id`. Strict Row-Level Security (RLS) is a mandatory architectural standard.
- **Secrets Management**: AWS secrets must always be fetched from `secrets.aivibe.cloud`.
- **App Registrations**: Each new app must be registered in the central database with an `app_id` assigned to the default company entity (Aivibe & Aivedha).
- **Instant Signup Plans**: Every user signup must automatically be assigned to the free plan for all registered `app_id`s in the database, enabling immediate multi-platform trial access.

## 14. Poetic HTML Email Templates (SES Only)
All automatic alerts, notifications, and transactional emails must be configured with poetic, rich, attractive, and colorful HTML templates. Emails must use SES ready-to-use domains in AWS and must use standardized sender addresses (e.g., `no-reply@...` or `alerts@...`).

## 15. Transparent Pages & 3D Background Animations
UIs must feature a 3D animated background with transparent pages overlaid, giving a futuristic sense of depth and motion. When adding colors or accents, prioritize using a rich, vibrant shine of **Lime Green** or a rich, vibrant shine of **Pure Red**.

## 16. Stylish Top-Right Breadcrumbs
All pages must feature a stylish, clean breadcrumb navigation positioned on the top-right of the available empty space.

## 17. AWS-Style Precision Cards (Sharp Blades + Soft Shadow)
Visual borders of components and cards must have thin, sharp, blade-like outer outlines with extremely clean borders (resembling the cards on the official AWS website), blended with soft, elegant shadows that perfectly match the application's overall color theme. (Maintain the rounded corners specified in Rule 2 while ensuring card edges and thin outlines are precise and sharp-bladed).

## 18. Latest Dependency Versions Only
Always use the latest stable versions of all dependencies and packages. There is zero tolerance for skipping, delaying, or staying on older versions due to outstanding minor bugs or unfixed issues. Solve outstanding issues or adapt code, but keep packages fully up-to-date.

## 19. Complete Code-First (No Stories, Minimal Text)
Never rush to build or deploy. Complete 100% of the code architecture before attempting compilation or deployment. Keep all responses, explanations, and summaries to a single line. Avoid writing long narratives, documentation blocks, or stories. Focus strictly on clean, functional, running code.

## 20. Comprehensive Event Fault Tolerance (At least 2 Positive, 2 Negative)
Every single event, action, or page load function must feature bulletproof exception-handling. The implementation must explicitly handle and manage:
- At least **two** positive, fully successful paths.
- At least **two** negative, broken, interrupted, or offline scenario paths.
No component should crash or hang when a network or state failure occurs.

## 21. Case-Sensitive Naming Conventions
Follow case-sensitive, fully unified naming conventions across the frontend and backend. Always compile a naming-registry mapping list of variables, functions, and columns to eliminate odds or mismatches and keep the entire stack absolutely uniform.

## 22. 98%+ Grounding Confidence (No Hallucinations, Online Research)
Never judge prompts, assume facts, guess values, or hallucinate. Grounding confidence must stay at 98% and above for every single action. The model must actively search online for up-to-date information, libraries, APIs, and guidelines matching the specific context, and never deviate from the established codebase conventions unless executing a planned, 100% guaranteed improvement in both visual and functional aspects.
