---
name: guide-specs
description: Exhaustively interview the user about a feature, plan, or architectural decision before implementation. Use when asked to grill specs, stress-test a design, or eliminate ambiguity before coding.
---

# Guide Specs

Resolve every applicable design decision with the user before implementation. Finding facts is your responsibility; making decisions is the user's. Never treat recommendations, unanswered questions, or hidden assumptions as approval.

1. **Investigate and map decisions.**

   Inspect existing code, architecture, conventions, interfaces, and dependencies before asking questions. Use sub-agents for independent investigations when available. Distinguish **OBSERVED** facts, **PROPOSED** designs, and **CONFIRMED** decisions.

   Map decisions as a dependency tree, recursively covering interfaces, data ownership, architectural consequences, edge cases, and failure modes. Mark inapplicable branches explicitly. Treat pending investigations as prerequisites without blocking unrelated questions.

2. **Interview in rounds.**

   The **frontier** is every unresolved decision whose prerequisites are settled. In each round:

   - Ask the entire frontier using numbered questions, options, recommendations, reasoning, implementation impact, and dependencies.
   - Wait for answers; do not ask dependent questions before their prerequisites are resolved.
   - Update the tree and blueprint, examining new abstractions, data structures, and flows for further decisions.
   - Recompute the frontier and repeat until no unresolved decisions or hidden implementation assumptions remain.

   Use ASCII-safe formatting:

   ```text
   Q1: Should the feature reuse existing state or own its state?
   A. Reuse existing state
   B. Introduce a dedicated manager
   Recommended: A
   Reason: Reduces duplication and lifecycle complexity.
   Implementation impact: Modify the state manager and SessionState interface.
   Dependencies: None
   ```

3. **Maintain a living blueprint.**

   After each round, show a concise, technically specific blueprint containing:

   - **Decisions:** Confirmed, unresolved, blocked, and pruned.
   - **Scope:** Included and excluded behavior.
   - **Architecture:** Components, responsibilities, and interactions.
   - **Code and schemas:** Proposed signatures, structs, interfaces, and useful snippets. Prefer JSON for serialized data and native types for project-specific structures.
   - **Execution:** Control flow, state transitions, errors, and edge cases.
   - **Files:** A hierarchical tree labeled NEW, MODIFY, or REMOVE, with each change explained:

     ```text
     src/
       services/
         session.ts       [MODIFY] - Extend lifecycle
         registry.ts      [NEW]    - Manage active sessions
       types/
         session.ts       [MODIFY] - Add state interface
       api/
         sessions.ts      [MODIFY] - Expose endpoint
     ```

   Label tentative details PROPOSED and include concrete examples where useful.

4. **Obtain implementation approval.**

   When the frontier is empty, audit the whole tree for hidden assumptions, unexplored consequences, and unresolved dependencies. Resume interviewing if any emerge.

   Otherwise, present the final specification and blueprint, including file changes, schemas, and a validation plan. Ask:

   **"All applicable design branches are resolved. Do you approve implementation?"**

   Wait for explicit approval before implementing.

5. **Implement and handle drift.**

   Capture the Git baseline and existing working-tree changes, then implement autonomously against the approved blueprint.

   Pause affected work before any material deviation: new abstractions or flows; unplanned files, dependencies, or persistent state; changed interfaces, schemas, or behavior; or unexpected complexity or incompatibility.

   Explain the approved design, discovered constraint, alternatives, recommendation, and file/code impact. Resume once the user resolves the issue. This is a targeted clarification, not another global approval gate; routine details that preserve the design need no interruption.

6. **Validate and report.**

   Run available tests, type checks, builds, and relevant validation. Audit source-code changes against the captured baseline using `git diff --shortstat`, excluding tests, configuration, generated files, and dependency manifests as appropriate. Account for new untracked source files separately and exclude pre-existing user changes.

   Report:

   - Implemented behavior versus the approved specification.
   - Actual file tree and changes, mapping confirmed decisions to files or symbols.
   - Approved deviations and unresolved concerns.
   - Validation results.
   - Production files changed, lines added/removed/net, new abstractions, new dependencies, and unplanned execution flows.
   - Complexity assessment: branching, coupling, duplication, interface growth, unnecessary abstractions, and simplification opportunities.

   Prioritize correctness and maintainability over fewer lines.
