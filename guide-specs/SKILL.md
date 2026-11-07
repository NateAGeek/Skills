---
name: guide-specs
description: Exhaustively interview the user about a feature, plan, or architectural decision before implementation. Use when asked to grill specs, stress-test a design, or eliminate ambiguity before coding.
---

# Grill Specs

Interview the user relentlessly until every applicable design decision is explicitly resolved. Do not silently assume implementation details.

## 1. Build the design tree

Map decisions as a dependency tree. Each decision may introduce new dependent decisions.

Work in **rounds**. The **frontier** contains every unresolved decision whose prerequisites are settled.

- Ask the entire frontier in each round.
- Number questions and provide recommended answers with reasoning.
- Never ask questions dependent on unresolved questions in the same round.
- After each response, update the tree and recompute the frontier.
- Visit every applicable branch recursively, including edge cases, failure modes, interfaces, data ownership, and architectural consequences.
- Mark inapplicable branches explicitly rather than silently ignoring them.
- Never treat recommendations or unanswered questions as approved decisions.

The interview ends only when no unresolved decisions or hidden implementation assumptions remain.

## 2. Investigate independently

Finding facts is your responsibility; making decisions is the user's.

Inspect existing code, architecture, conventions, interfaces, and dependencies before asking questions. Use sub-agents for independent investigations when available.

Do not block unrelated questions on pending investigations. Treat unresolved discoveries as prerequisites and continue with the available frontier.

Distinguish **OBSERVED**, **PROPOSED**, and **CONFIRMED** information.

## 3. Question format

Use ASCII-safe formatting (no Unicode symbols).

```text
Q1: State ownership

Should the feature reuse existing state or introduce its own?

A. Reuse existing state
B. Introduce a dedicated manager

Recommended: A
Reason: Reduces duplication and lifecycle complexity.

Implementation impact:
- Modify existing state manager
- Extend SessionState interface

Dependencies: None
```

Wait for the user's answers before advancing.

## 4. Maintain a living implementation blueprint

After each round, show what you currently believe will be implemented, updating it as decisions change.

Include:

- **Decisions:** Confirmed, unresolved, blocked, and pruned.
- **Architecture:** Components, responsibilities, and interactions.
- **Code:** Proposed function signatures, structs, interfaces, and relevant implementation snippets.
- **Schemas:** Prefer JSON for serialized data; use native types for project-specific structures.
- **Execution:** Control flow, state transitions, error paths, and edge cases.
- **Files:** Hierarchical tree of NEW, MODIFY, and REMOVE files, with explanations.
- **Scope:** Explicitly included and excluded behavior.

Example:

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

Show concrete examples where useful. Identify anything tentative as PROPOSED. Keep the blueprint concise but technically specific.

Any newly proposed abstraction, data structure, or execution flow must be examined for additional design decisions.

## 5. Single approval gate

When the frontier is empty, audit the entire tree for hidden assumptions, unvisited consequences, and unresolved dependencies.

If new decisions emerge, continue interviewing.

Otherwise present the final specification, implementation blueprint, file changes, schemas, and validation plan.

Ask:

**"All applicable design branches are resolved. Do you approve implementation?"**

Do not implement before explicit approval.

## 6. Implement and detect drift

After approval, implement autonomously according to the confirmed specification.

Continuously compare actual changes against the approved blueprint.

If implementation requires a material deviation, such as:

- New architectural abstractions or execution flows.
- Unplanned files, dependencies, or persistent state.
- Changed interfaces, schemas, or behavior.
- Unexpected complexity or incompatible existing code.

**Stop the affected work and ask for refinement before implementing the deviation.**

Explain the original design, discovered constraint, proposed alternatives, recommendation, and file/code impact.

This is a targeted clarification, not another global approval gate. Resume once resolved.

Do not interrupt for routine implementation details that preserve the approved design.

## 7. Validate and audit complexity

Run available tests, type checks, builds, and relevant validation.

Capture Git baseline and working-tree changes before implementation. After implementation, calculate source-code-only changes relative to that baseline.

Use `git diff --shortstat` with repository-appropriate exclusions for tests, configuration, generated files, and dependency manifests. Include new untracked source files separately; do not attribute pre-existing user changes to the implementation.

Report:

```text
SOURCE CHANGE AUDIT

Production files changed:  5
Lines added:              142
Lines removed:             87
Net lines:                +55

New abstractions:           2
New dependencies:           0
Unplanned execution flows:  0
```

Review complexity beyond line counts: branching, coupling, duplicated logic, interface growth, unnecessary abstractions, and opportunities to simplify.

Do not optimize for fewer lines at the expense of correctness or maintainability.

## 8. Final report

Summarize:

- Implemented behavior versus approved specification.
- Actual file tree and changes.
- Confirmed decisions mapped to implementing files or symbols.
- Approved deviations and unresolved concerns.
- Test/build results.
- Filtered Git shortstat and architectural complexity assessment.

**Core rules:** Exhaust every design branch. Never assume user decisions. Show concrete implementation plans as the interview progresses. Obtain one approval. Escalate material drift. Audit the result.
---
