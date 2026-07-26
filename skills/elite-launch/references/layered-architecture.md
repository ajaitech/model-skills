# Layered architecture

Preserve the repository's established clean boundaries; do not impose a new folder
shape when the project already has an equivalent owner.

## Responsibility flow

| Layer | Owns | Must not own |
|---|---|---|
| UI/presentation | Screens, reusable components, pure input validation, loading/empty/error/success states, guided recovery, accessibility | Raw SDK/network/database calls or duplicated business rules |
| Domain | Pure typed business rules, state transitions, calculations, invariants | Framework UI, network, filesystem, database, or global mutable state |
| Integration/API | One typed entry surface that routes calls to the correct transport/trust owner | Presentation markup or duplicated domain decisions |
| Data/infra | Persistence adapters, cache/storage, runtime clients, configuration resolution, observability wiring | User-facing behavior or hardcoded environment values |

## Integration branches

Use existing project names when available. If a project needs explicit branches,
separate:

| Branch | Boundary |
|---|---|
| AppSync/GraphQL | Application-owned queries, mutations, subscriptions |
| Lambda/REST | Application-owned service endpoints and invocations |
| Universal/platform | Shared SSO, organization, tenant, plan, billing, credit, and platform APIs |
| WebSocket/realtime | Connection lifecycle, channels, retry, ordering, and backpressure |
| External L1 | Government or public-authority integrations |
| External L2 | Non-government commercial/private integrations |

Every caller uses the typed integration surface; it must not reach into transport
internals. Each transport owner handles request typing, response validation, error
mapping, retry/idempotency policy, cancellation, and observability.

## Change gate

- One owner per responsibility; search before adding.
- Explicit request/response and persisted datatypes.
- Pure validators and domain rules reused across callers.
- No UI-owned integration calls or infrastructure-owned UI decisions.
- No client-authoritative price, identity, permission, or sensitive persisted fact.
- All affected states, events, cleanup, migrations, and regression consumers covered.
- No production mocks, sample data, hidden fallback, duplicate path, or dead branch.
