---
name: documentation-process
description: Best practices for writing and maintaining user-facing documentation — tone, structure, type tables, JSDoc as source of truth, and what to keep out of docs. Use this skill whenever writing, editing, or reviewing .mdx documentation pages, updating JSDoc on exported types, adding new docs pages, documenting new features or APIs, refactoring docs after code changes, or discussing documentation conventions. Also use when creating or updating meta.json navigation files, README files for examples, or protocol/wire-format docs.
---

## Overview

This skill defines documentation conventions for monorepos using Fumadocs with auto-generated type tables. All user-facing docs live in `docs/content/docs/` as `.mdx` files rendered by Fumadocs. Type reference tables are auto-generated from source TypeScript via `AutoTypeTable`, making JSDoc comments the single source of truth for type documentation.

The guiding principle: **docs describe what the user can do and when to use it. They never expose internal implementation details.**

---

## Documentation Stack

| Component | Location | Purpose |
|---|---|---|
| Fumadocs (MDX) | `docs/content/docs/**/*.mdx` | User-facing documentation pages |
| `AutoTypeTable` | `fumadocs-typescript/ui` | Auto-generates type reference tables from source `.ts` files |
| `source.config.ts` | `docs/source.config.ts` | Configures `remarkAutoTypeTable` plugin with a shared generator |
| `mdx.tsx` | `docs/components/mdx.tsx` | Registers `AutoTypeTable` as an MDX component with the generator |
| `meta.json` | `docs/content/docs/**/meta.json` | Controls page ordering and navigation labels per directory |

### Generator and Cache

Both `source.config.ts` and `mdx.tsx` create a `generator` instance with a filesystem cache at `.next/fumadocs-typescript`. This generator resolves TypeScript types at build time. You don't need to touch these files unless changing the cache location or adding custom type resolution logic — just use `<AutoTypeTable>` in your MDX and the existing pipeline handles the rest.

---

## Golden Rule: No Internal Implementation Details

User-facing documentation must never mention:

- **Internal libraries or SDKs** — never reference underlying SDK package names or internals. The user interacts with your project's APIs, not the libraries powering them.
- **Internal delegation or execution paths** — never describe how one method is implemented in terms of another (e.g., "call() delegates to stream() internally").
- **Internal lifecycle mechanics** — don't enumerate internal execution order or hook pipelines in feature docs. Reserve that for dedicated reference pages.
- **Internal type names from dependencies** — avoid surfacing types from third-party packages in prose unless the user directly interacts with them.

### What to write instead

Focus on:
- **What the method does** from the user's perspective
- **When to use it** — decision criteria for choosing between options
- **What the user receives** — return types, events, result shapes
- **Code examples** showing real usage

### Examples

```markdown
<!-- BAD: exposes internals -->
`stream()` is the single execution path. It runs the full hook lifecycle
(alter message, before call, execute/LLM, after call, alter response),
manages conversation history, and yields typed events.

`call()` delegates to `stream()` internally — it consumes all events
silently and returns the `done` result.

<!-- GOOD: user-focused -->
Use `stream()` to receive events as they arrive from the model. It yields
typed events — `text`, `tool-call`, `tool-result`, `step-start` — followed
by a final `done` event containing the complete result.

Use `call()` when you only need the final result and don't need to process
events as they arrive.
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

Every type/interface reference in docs must use `AutoTypeTable` instead of hand-written markdown tables. This ensures type documentation stays in sync with source code automatically.

### Format

```mdx
## ConfigType

The configuration object passed to the factory function.

<AutoTypeTable path="../packages/core/src/module/module.types.ts" name="ConfigType" />
```

### Rules

1. **One `AutoTypeTable` per type** — each type gets its own `##` heading with a one-line description above the table.
2. **`path` is relative to the `docs/` directory** — always starts with `../packages/...`.
3. **`name` matches the exported TypeScript type name exactly** — case-sensitive.
4. **Never duplicate type information in prose** that `AutoTypeTable` already renders. Add a brief intro sentence, then the table.
5. **If a type table was previously a manual markdown table, replace it** with `AutoTypeTable`.

### When NOT to use AutoTypeTable

`AutoTypeTable` is for documenting TypeScript interfaces and type aliases. Don't use it for:

- **Function-level or registry APIs** — when documenting a registry, service, or set of functions (not a single type), use prose and code examples instead. The API shape is better communicated through usage patterns than a type table.
- **Wire protocol / message format docs** — daemon or server protocols with request/response shapes are often clearer as manual markdown tables with columns like `Field | Type | Description`.
- **Feature comparison tables** — conceptual overviews comparing options.
- **Example file listings** — README tables listing example scripts.
- **Conceptual overviews** that don't map to a single TypeScript type.

---

## JSDoc Is the Source of Truth

Since `AutoTypeTable` renders directly from TypeScript source, **JSDoc comments on types and interfaces are the actual documentation the user reads**. This means:

### JSDoc quality requirements

1. **Every exported interface and type gets a JSDoc block** explaining its purpose.
2. **Every field gets a single-line `/** */` comment** describing what it does from the user's perspective.
3. **Optional fields include `@default`** when there is a meaningful default.
4. **`@example` blocks** on factory functions and major interfaces.
5. **`@internal` tag** on implementation-only API surface.

### JSDoc must follow the same "no internals" rule

Since JSDoc renders in the docs, it must not reference:
- Internal SDK names or package internals
- Internal implementation details
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
 * Custom execute override — replaces the default behavior with arbitrary logic.
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
title: createSomething
description: Factory function for creating X with Y capabilities.
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
4. **Feature sections** — one per capability
5. **Related links** — cross-references to other docs pages

### Concept / Guide pages

Not every page documents an API. For concept pages (overviews, guides, getting-started):

```mdx
---
title: {Concept Name}
description: {One-sentence summary.}
---

# {Concept Name}

{2-3 sentence intro explaining the concept and why it matters.}

## {Sub-topic}

{Explanation with code examples.}

## {Sub-topic}

...
```

No `AutoTypeTable` needed — these pages use prose and code examples exclusively.

---

## Navigation: `meta.json` Files

Each directory under `docs/content/docs/` has a `meta.json` that controls page ordering and navigation labels. When adding a new docs page:

1. **Create the `.mdx` file** in the appropriate directory.
2. **Add its slug to the `meta.json`** in the same directory so it appears in navigation.
3. **Position it logically** — overview/index pages first, then core concepts, then advanced/reference.

If creating a new directory (e.g., a new feature area), also create a `meta.json` inside it and reference the directory from the parent `meta.json`.

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

## Documentation Patterns by Domain

Different parts of the codebase call for different documentation approaches:

### API / Factory function docs
Use the standard page layout pattern with `AutoTypeTable` for config and return types. This covers most core APIs (agents, flows, tools, hooks, credentials, strategy).

### Protocol / Wire format docs
Use manual markdown tables with `Field | Type | Description` columns. These document message shapes over the wire, not TypeScript interfaces the user imports. Keep request and response tables visually distinct.

### CLI / Server docs
Focus on commands, flags, and configuration. Use code blocks for command examples. Document environment variables and config file formats. Type tables are rarely useful here — prefer structured prose.

### Built-in implementations
When documenting built-in tools, hooks, flows, or other implementations: lead with what it does and a usage example, then document any specific config options. Keep these pages short — the user wants to know how to use it, not how it works internally.

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

1. All type references use `AutoTypeTable`, not manual markdown tables (unless non-type content or function/registry APIs).
2. No mentions of internal SDK names or third-party package internals in user-facing text.
3. No internal implementation details (delegation, execution paths, lifecycle ordering in feature docs).
4. JSDoc on source types matches the behavior described in `.mdx` prose.
5. No stale JSDoc that describes removed or changed behavior.
6. Code examples are complete and runnable (imports included on first use).
7. `bun run build` in `docs/` succeeds (validates `AutoTypeTable` paths and type names).
8. Cross-references use relative Fumadocs paths, not absolute URLs.
9. New pages are added to the relevant `meta.json` for navigation.
10. `meta.json` ordering is logical (overview first, then concepts, then reference).
