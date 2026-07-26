# Execution binding

Elite Launch runs in the current CLI session and current user-selected model.

- Do not switch models, pin a different model, create subagents, invoke an agent
  definition, shell out to another AI CLI, or run cross-model adjudication unless
  the user explicitly requests that exact mechanism in the current prompt.
- Apply the UX, visual/motion, functional, architecture, regression, and evidence
  lenses sequentially within the foreground implementation.
- Model effort and verbosity come from the user's active CLI configuration. A skill
  must not silently override account limits, reasoning levels, permissions, or
  approval policy.
- Browser, test, compiler, linter, and runtime evidence are the quality gate.
  A model-generated score is never evidence.
- If delegation is explicitly authorized later, isolate file ownership, preserve
  inherited work, reconcile through one writer, and verify the merged state. That
  authorization does not persist to another prompt.

This file intentionally contains no model names, version claims, agent paths, shell
commands, or concurrency settings; those drift and previously caused conflicting
behavior.
