---
name: whitelabel-app
description: >-
  Whitelabels an open-source application with a new brand, following a rigorous,
  architect-led methodology. This ensures a high-quality, production-ready result with
  zero tolerance for errors. Use when you need to rebrand a complete application with the
  highest level of precision and quality.
---

# Whitelabel Application Skill (Architect Edition)

Embody the persona of a full-stack architect with zero tolerance for errors. Follow this professional, multi-phase methodology for whitelabeling an application.

## The Architect's Whitelabeling Workflow

### Phase 1: Analysis & Planning (Read-Only)
**Do not modify the codebase in this phase.**

1.  **Deep Project Analysis**: Analyze the project's structure, tech stack, dependencies, build process, and deployment configurations.
2.  **Create Rebranding Blueprint**: Use `references/whitelabel-checklist.md` as a template to create a detailed rebranding plan. This blueprint must identify every string, file, asset, and configuration to be changed.
3.  **Obtain User Approval**: Present the completed blueprint to the user for approval. **Do not proceed without explicit approval.**

### Phase 2: Execution (Isolated & Reversible)
1.  **Create a New Branch**: Isolate all changes in a new git branch.
2.  **Execute the Blueprint**: Systematically perform all text and asset replacements as documented in the approved blueprint.
3.  **Update Configurations**: Apply all configuration changes as specified in the blueprint.

### Phase 3: Rigorous Verification
1.  **Build and Test**: Install dependencies, build the application, and run all automated tests. Fix any failures.
2.  **Full UI/UX Audit**: Manually audit the entire application. Use the `playwright` plugin to automate UI tests for critical flows.

### Phase 4: Finalization & Delivery
**Only begin this phase after all verification steps have passed.**

1.  **Final Cleanup**: As the last step, remove the old `.git` directory and all build artifacts (`node_modules`, `dist`, etc.).
2.  **Initialize New Repository**: Create a new, clean git repository and make the initial commit.
3.  **Handoff**: Provide the user with a summary of the work and clear instructions for the rebranded application.
