---
name: documentation-process
description: Best practices for writing and maintaining user-facing documentation — tone, structure, type tables, JSDoc as source of truth, and what to keep out of docs
---

## Overview

This skill defines the documentation conventions for this monorepo. All user-facing docs live in `docs/content/docs/` as `.mdx` files rendered by Fumadocs. Type reference tables are auto-generated from source TypeScript via `AutoTypeTable`, making JSDoc comments the single source of truth for type documentation.

The guiding principle: **docs describe what the user can do and when to use it. They never expose internal implementation details.**

---

## Documentation Stack

| Component | Location | Purpose |
|---|---|---|
| Fumadocs (MDX) | `docs/content/docs/**/*.mdx` | User-facing documentation pages |
| `AutoTypeTable` | `fumadocs-typescript/ui` | Auto-generates type reference tables from source `.ts` files |
| `source.config.ts` | `docs/source.config.ts` | Configures `remarkAutoTypeTable` plugin |
| `mdx.tsx` | `docs/components/mdx.tsx` | Registers `AutoTypeTable` as an MDX component |

---

## Golden Rule: No Internal Implementation Details

User-facing documentation must never mention:

- **Internal libraries or SDKs** — never say "AI SDK", "Vercel AI SDK", or reference `ai` package internals. The user interacts with Comma Agents APIs, not the underlying SDK.
- **Internal delegation or execution paths** — never say "call() delegates to stream() internally", "single execution path", or describe how one method is implemented in terms of another.
- **Internal hook lifecycle mechanics** — don't enumerate the hook execution order (alter message → before call → execute → after call → alter response) in feature docs. That belongs in the Hooks reference page only.
- **Internal type names from dependencies** — avoid surfacing types like `LanguageModel`, `StepResult`, `ModelMessage` in prose unless the user directly interacts with them.

### What to write instead

Focus on:
- **What the method does** from the user's perspective
- **When to use it** — decision criteria for choosing between options
- **What the user receives** — return types, events, result shapes
- **Code examples** showing real usage

### Examples

```markdown
<!-- BAD: exposes internals -->
`stream()` is the single execution path for all agents. It runs the full
hook lifecycle (alter message, before call, execute/LLM, after call, alter
response), manages conversation history, and yields typed events.

`call()` delegates to `stream()` internally — it consumes all events
silently and returns the `done` result.

<!-- GOOD: user-focused -->
Use `stream()` to receive events as they arrive from the model. It yields
typed `AgentStreamEvent` values — `text`, `tool-call`, `tool-result`,
`step-start` — followed by a final `done` event containing the complete result.
```

```markdown
<!-- BAD: references internal SDK -->
Return the conversation history as AI SDK messages.
The model is resolved via the AI SDK provider system.

<!-- GOOD: abstracted -->
Return the conversation history as structured messages.
The model is resolved via the model registry and provider system.
```

---

## Type Tables: Use `AutoTypeTable`, Not Manual Markdown

Every type reference in docs must use `AutoTypeTable` instead of hand-written markdown tables. This ensures type documentation stays in sync with source code automatically.

### Format

```mdx
## AgentConfig

The configuration object passed to `createAgent`.

<AutoTypeTable path="../packages/core/src/agents/agent/agent.types.ts" name="AgentConfig" />
```

### Rules

1. **One `AutoTypeTable` per type** — each type gets its own `##` heading with a one-line description above the table.
2. **`path` is relative to the `docs/` directory** — always starts with `../packages/...`.
3. **`name` matches the exported TypeScript type name exactly** — case-sensitive.
4. **Never duplicate type information in prose** that `AutoTypeTable` already renders. Add a brief intro sentence, then the table.
5. **If a type table was previously a manual markdown table, replace it** with `AutoTypeTable`.

### When manual tables are acceptable

Manual markdown tables are acceptable for non-type content:
- Wire protocol message references (daemon protocol docs)
- Feature comparison tables
- Example file listings (README tables)
- Conceptual overviews that don't map to a single TypeScript type

---

## JSDoc Is the Source of Truth

Since `AutoTypeTable` renders directly from TypeScript source, **JSDoc comments on types and interfaces are the actual documentation the user reads**. This means:

### JSDoc quality requirements

1. **Every exported interface and type gets a JSDoc block** explaining its purpose.
2. **Every field gets a single-line `/** */` comment** describing what it does from the user's perspective.
3. **Optional fields include `@default`** when there is a meaningful default.
4. **`@example` blocks** on factory functions and major interfaces.
5. **`@internal` tag** on implementation-only API surface (e.g., `appendHook`).

### JSDoc must follow the same "no internals" rule

Since JSDoc renders in the docs, it must not reference:
- Internal SDK names ("AI SDK", "Vercel AI SDK")
- Internal implementation details ("delegates to stream()", "wraps generateText")
- Stale behavior (e.g., "streaming is not supported" when it now is)

```ts
// BAD — leaks internal SDK
/** Get conversation context as AI SDK messages. */
getContext?(): ConversationContext;

// GOOD — user-focused
/** The conversation context — all turns, messages, and context management. */
getContext?(): ConversationContext;
```

```ts
// BAD — stale, references removed limitation
/**
 * Custom execute override.
 * Streaming is not supported when `execute` is set — calling `stream()` will throw.
 */

// GOOD — accurate, no stale claims
/**
 * Custom execute override — replaces the LLM call with arbitrary logic.
 * When set, `model` is not required.
 */
```

### Keep JSDoc and docs pages in sync

When refactoring changes behavior:
1. Update the JSDoc on the source type **first** — this is what `AutoTypeTable` renders.
2. Update the `.mdx` prose only if the surrounding explanation needs to change.
3. Never let a docs page describe behavior that contradicts the JSDoc.

---

## MDX Page Structure

### Frontmatter

Every `.mdx` file starts with YAML frontmatter:

```yaml
---
title: createAgent
description: Factory function for creating LLM-backed agents with string-based model and tool resolution.
---
```

- `title` — the API name or concept name
- `description` — one sentence, used in navigation and search

### Page layout pattern

Follow this structure for API reference pages:

```mdx
---
title: {API Name}
description: {One-sentence summary.}
---

# {API Name}

{1-2 sentence intro explaining what this does and why you'd use it.}

{Quick-start code example showing the most common usage.}

## {ConfigType}

{One-line description.}

<AutoTypeTable path="..." name="{ConfigType}" />

## {ReturnType}

{One-line description of what the factory/function returns.}

<AutoTypeTable path="..." name="{ReturnType}" />

## {Feature Section}

{Explain the feature from the user's perspective. Show code examples.}

## {Another Feature Section}

...
```

### Section ordering

1. **Intro + quick-start example** — always first
2. **Config/input types** — what you pass in
3. **Return types** — what you get back
4. **Feature sections** — one per capability (streaming, cancellation, hooks, etc.)
5. **Related links** — cross-references to other docs pages

---

## Prose Style

### Tone

- **Direct and concise** — no filler words, no marketing language.
- **Second person ("you")** when addressing the user: "you never need to configure providers directly."
- **Present tense** — "returns", "yields", "accepts" (not "will return").
- **No superlatives or praise** — never "powerful", "elegant", "seamless".

### Describing methods and APIs

- Lead with **what it does**: "Send a message and get a complete response."
- Follow with **when to use it**: "Use this when you need the final result and don't need to process events as they arrive."
- End with **technical details** only if relevant: "Returns an `AbortablePromise` — call `.abort()` to cancel."

### Code examples

- **Every feature section needs at least one code example.**
- Examples use `ts` code blocks with real, runnable code.
- Import statements included when showing a feature for the first time on the page.
- Comments in code examples should be brief — `// final text response`, not `// This is the final text response returned by the agent after processing`.

---

## Cross-References

Use relative Fumadocs links:

```markdown
See the [Hooks documentation](/docs/core/hooks) for the full hook API.
```

Never use absolute URLs for internal docs links. Never link to source code files from user-facing docs.

---

## README Files (Examples)

Example READMEs (`examples/*/scripts/README.md`) follow different rules than API docs:

1. **Prerequisites section** — lists install steps and env vars.
2. **Running section** — exact commands to run examples.
3. **Examples table** — `| # | File | Description |` format with concise descriptions.
4. **Key APIs Used** — flat list of the APIs demonstrated.
5. **Same "no internals" rule applies** — say "Install a provider package", not "Install an AI SDK provider package".

---

## Checklist: Before Merging Documentation Changes

1. All type references use `AutoTypeTable`, not manual markdown tables (unless non-type content).
2. No mentions of "AI SDK", "Vercel AI SDK", or internal SDK type names in user-facing text.
3. No internal implementation details (delegation, execution paths, hook ordering in feature docs).
4. JSDoc on source types matches the behavior described in `.mdx` prose.
5. No stale JSDoc that describes removed or changed behavior.
6. Code examples are complete and runnable (imports included on first use).
7. `bun run build` in `docs/` succeeds (validates `AutoTypeTable` paths and type names).
8. Cross-references use relative Fumadocs paths, not absolute URLs.
