---
name: elite-launch
description: Production-first execution doctrine for building, fixing, refactoring, integrating, reviewing, or shipping application code. Use for any development task that must be grounded in the real repository, complete across affected layers, reuse existing owners, avoid mocks and hardcoding, analyze scenarios, and finish with evidence and zero errors or warnings.
---

# Elite Launch

Execute the user's authorized outcome in the current foreground session. The
latest explicit prompt is the scope authority. Do not judge, reinterpret, expand,
or replace it. Do not create subagents, call another model, or launch an external
AI CLI unless the user explicitly requests that mechanism in the current prompt.

## Ground before editing

1. Resolve the canonical repository and inherited working-tree state.
2. Read project guidance and the exact project-knowledge document referenced
   there; validate affected entries against current source before relying on them.
3. Trace the real entry points, routes, UI/state, domain rules, typed integration,
   persistence/external effects, events, observability, recovery, and deployment
   boundary touched by the request.
4. Search for existing files, functions, components, validators, models, clients,
   configuration, and tests before creating anything.
5. Treat code and runtime evidence as current-state truth. Documents define
   requirements unless freshly generated from source.

## Production implementation

- Implement the complete authorized vertical slice directly in canonical files.
- No MVP, demo, sample, placeholder, visual-only control, fabricated response,
  production mock, silent success fallback, swallowed failure, or knowingly
  incomplete layer.
- Runtime identities, credentials, endpoints, business records, prices, and
  environment-specific values come from their existing configuration or data
  owner. Define protocol constants once in the correct typed owner.
- Reuse or refactor the current owner. Do not create parallel implementations,
  duplicate functions, shadow files, or `new`/`copy`/`final`/`fixed` variants.
- Preserve unrelated inherited changes. Remove only dead code made obsolete by
  the authorized change after proving it has no remaining consumers.
- Put task scratch data only under the active project's `.tmp/<tool>/<task>/` and
  remove it after verification.

Read [references/layered-architecture.md](references/layered-architecture.md)
before changing boundaries. For UI work, run
`ui-flow-review-loop`. For motion, read
[references/animation-systems.md](references/animation-systems.md). Load other
specialists only through
[references/specialty-support.md](references/specialty-support.md).

## Scenario and regression matrix

Analyze the affected events before editing: initial/loading/empty/partial/success/
failure; invalid input; repeated action; stale response; cancellation; interruption
and resume; offline/timeout/disconnect/reconnect/retry; idempotency; concurrency and
races; dependency failure; lifecycle restoration; migration/backward compatibility;
localization; accessibility; large data; performance; and resource cleanup. Add
domain-specific cases found in the code.

## Verification and closure

Use focused checks while implementing. Run the full build only after the coherent
end-to-end slice is complete. Then run every project-required format, lint, static
analysis, type, unit, integration, end-to-end, build, runtime, browser, console,
network, accessibility, and performance gate applicable to the change. Fix every
error and warning, rerun from the settled source, and update the project-knowledge
document from the final source.

Report only exact changed artifacts, verification evidence, and genuine blockers.
Do not replace implementation with advice or spend tokens on a progress story.

Source: https://github.com/ajaitech/model-skills/tree/main/skills/elite-launch
