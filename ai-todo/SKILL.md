---
name: ai-todo
description: Find and resolve code comments labeled `AI:` by inventorying them, grouping them by change domain, and preparing an approval-gated implementation plan. Use when a user asks to process AI-specific TODOs or address `AI:` comments in a codebase.
---

# AI TODO

Treat comments containing the exact label `AI:` as requests intended for the agent. The text after the label is the request; it does not need to contain the word `TODO`.

## Discover and assess

Search the entire user-scoped codebase for `AI:` in source comments. Exclude generated output, dependency/vendor directories, caches, and version-control metadata unless the user explicitly includes them.

Read enough surrounding code and relevant project guidance to understand every matching request. Build a complete inventory containing each request's location, intent, and affected behavior. Do not edit code during this phase.

Distinguish actionable requests from incidental matches, such as documentation explaining the label or string data that is not a source comment. Report incidental or already-satisfied items separately rather than silently treating them as implementation work.

## Resolve uncertainty

Identify requests that conflict with one another, conflict with current behavior or project guidance, lack a necessary product decision, or leave the intended control flow unclear. Ask focused questions about those points before proposing a final plan. Do not invent a resolution that materially changes the requested behavior.

## Propose the plan

Group the actionable items by domain of change, such as a feature area, subsystem, data model, API, or UI flow. Prefer domains that match the codebase's own boundaries over generic categories.

Present:

- the full inventory, grouped by domain;
- dependencies, conflicts, assumptions, and unanswered questions;
- a step-by-step implementation plan ordered by dependency and including appropriate verification; and
- any items that should not be implemented, with the reason.

Explicitly ask the user to approve or revise the plan. Stop after presenting it. Plan approval authorizes only the listed work and does not expand the task to unrelated cleanup.

## Implement after approval

Only after the user approves the plan, implement the approved items. If the user revises the plan, restate the resulting scope when needed and follow that scope. If implementation exposes a new conflict or requires a material decision not covered by the approval, pause and ask the user rather than guessing.

Preserve unrelated changes. Add or update tests when they are the appropriate verification for the affected behavior, and run focused checks in proportion to the changes. Remove or update each addressed `AI:` comment so completed work is not rediscovered as pending; preserve comments for deferred items and make their status clear when appropriate.

Finish by summarizing the implemented changes, verification results, and any deferred or unresolved `AI:` items.
