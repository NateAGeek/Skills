---
name: ts-patterns
description: TypeScript coding conventions — functional architecture, file organization, naming discipline, barrel exports, test philosophy, and JSDoc standards. Use this skill whenever writing or reviewing TypeScript code, creating new modules, refactoring existing code, setting up project structure, or discussing TypeScript best practices.
---

## Overview

This skill defines universal TypeScript conventions for any project. The architecture favors functional patterns over classes, strict naming discipline, domain-oriented file organization, and tests that serve as behavioral specifications.

These conventions are defaults. When a project has established patterns that conflict, follow the project's conventions and note the divergence. When starting fresh or no convention exists, apply these.

---

## Functional Architecture

Prefer factory functions over classes. Factories make dependencies explicit, avoid `this` binding pitfalls, and compose naturally. State lives in closures, not on instances.

### Factory structure

```ts
export function createRouter(config: RouterConfig): Router {
  const routes = new Map<string, RouteHandler>();
  let middleware: Middleware[] = config.middleware ? [...config.middleware] : [];

  function addMiddleware(handler: Middleware): void {
    middleware = [...middleware, handler];
  }

  const router: Router = {
    name: config.name,

    addRoute(pattern: string, handler: RouteHandler): void {
      routes.set(pattern, handler);
    },

    use(handler: Middleware): void {
      addMiddleware(handler);
    },

    async handle(request: IncomingRequest): Promise<Response> {
      const matchedRoute = matchRoute(routes, request.path);
      const transformedRequest = applyMiddleware(middleware, request);
      return executeHandler(matchedRoute, transformedRequest);
    },
  };

  return router;
}
```

### Internal helpers vs. utils

Only define helper functions inside the factory when they need to read or mutate closure state. If a function is pure -- it takes inputs and returns outputs without touching closure variables -- it belongs in `{domain}.utils.ts`.

```ts
// router.utils.ts — pure, no closure state needed
export function matchRoute(
  routes: Map<string, RouteHandler>,
  path: string,
): RouteMatch | undefined {
  // ...
}

export function applyMiddleware(
  middleware: Middleware[],
  request: IncomingRequest,
): IncomingRequest {
  // ...
}
```

```ts
// router.ts — inside the factory, because it mutates `middleware`
function addMiddleware(handler: Middleware): void {
  middleware = [...middleware, handler];
}
```

The test for where a function belongs: does it reference a variable declared in the factory's closure scope? If yes, it stays in the factory. If no, extract it to utils.

### Deconstruct Paramaters of Functions

```ts
// Bad -- unecessery function deconstruciton within the function
function readConfig(options: ConfigReadOptions) {
  const { config, source, destination } = options;
}

// Good -- keeps the desconstruction within the function and removes the const noise
function readConfig({ config, source, destination } : ConfigReadOptions) {
  // Logic can use the scoped vars from the deconstruction
}
```

### Keep function parameters minimal

Every function parameter must have a purpose in the function's current behavior or be required by a contract the function implements. Start with the smallest signature that satisfies the known requirement, and add a parameter only when production behavior actually needs input from the caller.

Do not add parameters:

- For hypothetical future features or behavior
- Because a caller might eventually need more control
- Only to make a test easier to write
- To expose internal implementation details to tests
- As unused placeholders, including underscore-prefixed parameters

Tests must exercise the production API that the behavior requires. They do not justify expanding that API. If testing is difficult, improve the design around real dependencies or test observable behavior rather than adding a test-only argument.

```ts
// Bad -- `now` exists only so a test can control the result
function isExpired(entry: CacheEntry, now: number): boolean {
  return now - entry.createdAt > entry.ttl;
}

// Good -- minimal signature for the current production behavior
function isExpired(entry: CacheEntry): boolean {
  return Date.now() - entry.createdAt > entry.ttl;
}
```

When variable behavior is a real product requirement, represent it as an explicit dependency with a domain-specific name. Do not invent that flexibility solely for tests.

```ts
interface CacheConfig {
  readonly getCurrentTime: () => number;
}
```

External callback, framework, and interface contracts are the exception. Include a parameter when the contract requires it; when TypeScript allows a callback to omit unused trailing parameters, omit them instead of naming placeholders.

### Why factories over classes

- **No `this` ambiguity** -- methods are plain functions, safe to destructure or pass as callbacks without `.bind()`
- **True privacy** -- closure variables are invisible, not just `private` keyword conventions
- **Config immutability** -- spread config arrays/objects into local copies so the caller's original is never mutated
- **Composition over inheritance** -- shared behavior lives in builder functions that multiple factories delegate to, not in base classes

### Don't hide conditionals behind one-off functions

If a conditional check is only used once, write it inline. Wrapping a simple boolean expression in a named function like `isValid()` or `shouldRetry()` just moves the logic out of sight without adding real clarity -- the reader now has to jump to a function definition to understand a straightforward check.

```ts
// Bad -- hides a simple check behind a function name
function isExpired(entry: CacheEntry): boolean {
  return Date.now() - entry.createdAt > entry.ttl;
}

if (isExpired(entry)) {
  cache.delete(key);
}

// Good -- the condition is right where you need to understand it
if (Date.now() - entry.createdAt > entry.ttl) {
  cache.delete(key);
}
```

Extract a conditional into a named function only when:
- It is reused in multiple places
- The expression is genuinely complex (multiple clauses, bit operations, non-obvious math) and a name helps the reader skip over the details
- You are comparing against the result of a computation that needs its own scope

### When classes might appear

Third-party libraries and framework integrations sometimes require classes (decorators, DI containers, ORM entities). That's fine -- use them at the boundary and keep your own domain logic functional.

---

## File Organization

All filenames use `kebab-case`. Each domain module follows a consistent file-per-concern layout:

| File | Purpose | Exported from barrel? |
|---|---|---|
| `index.ts` | Barrel -- only re-exports | IS the barrel |
| `{domain}.ts` | Factory functions, orchestration logic | Yes |
| `{domain}.types.ts` | All types for this domain | Yes |
| `{domain}.constants.ts` | Named constants, default value objects | No (internal) |
| `{domain}.utils.ts` | Pure stateless helpers | No (internal) |
| `{domain}.test.ts` | Co-located unit tests | No |
| `test.utils.ts` | Shared test fixtures for the module | No |

Every domain and subdomain follows this layout. No exceptions, no dangling files. If a file exists in a domain folder, it must fit one of these roles with the correct naming convention.

### Types live in `{domain}.types.ts`

Every type belonging to a domain goes in one file. This includes config interfaces, public contracts, result types, and callback/hook types. When someone asks "what types does the router module expose?", there's exactly one place to look.

A bare `types.ts` (no domain prefix) is reserved for truly cross-cutting primitives shared across unrelated domains. If a type is tightly coupled to a domain, it belongs in `{domain}.types.ts`.

### Domain folder example

```
router/
  router.ts              # createRouter() factory
  router.types.ts        # RouterConfig, Router, RouteMatch, Middleware
  router.constants.ts    # DEFAULT_TIMEOUT, HTTP_METHODS
  router.utils.ts        # matchRoute(), applyMiddleware(), parsePath()
  router.test.ts         # unit tests
  test.utils.ts          # shared test fixtures
  index.ts               # barrel
```

### Subdomains

When a domain has specialized variants, each gets its own subdirectory. Subdomains follow the exact same file organization as the parent domain -- every subdomain folder has its own `{domain}.ts`, `{domain}.types.ts`, `index.ts`, and whatever other files it needs from the standard layout.

```
parsers/
  parsers.ts
  parsers.types.ts
  index.ts
  json/
    json-parser.ts
    json-parser.types.ts
    json-parser.utils.ts
    json-parser.test.ts
    index.ts
  yaml/
    yaml-parser.ts
    yaml-parser.types.ts
    yaml-parser.test.ts
    index.ts
```

No loose files floating outside the convention. If a subdomain has types, they go in `{subdomain}.types.ts`. If it has helpers, they go in `{subdomain}.utils.ts`.

### Constants file

Constants are extracted into their own file for two reasons: they change rarely (stable imports), and they keep magic values out of logic code.

```ts
export const DEFAULT_TIMEOUT = 30_000;
export const MAX_REDIRECTS = 10;
```

Constants files are internal -- never re-exported from barrels.

---

## Barrel Exports

### Rules

1. **Only `export` and `export type` statements** -- no logic, no side-effect imports.
2. **Separate value exports from type exports** -- never mix in the same statement.
3. **Internal files are never exported**: utils, constants, tests, test fixtures.
4. **Internal-only types stay internal**: resolved-options types, infrastructure generics, implementation details.

### Format

```ts
export { createRouter } from "./router";

export type {
  Router,
  RouterConfig,
  RouteMatch,
} from "./router.types";
```

The barrel is the module's public API contract. If it's not exported from the barrel, it's an implementation detail that can change without notice.

---

## Naming Discipline

### The core rule: names communicate intent

Every variable, parameter, function, and type parameter must use full, descriptive words. The goal is that someone reading the code understands what something represents without looking at its declaration.

### Prohibited patterns

| Banned | Better | Reason |
|---|---|---|
| `fn`, `cb` | `onComplete`, `transformFn` | What kind of function? |
| `ctx` | `requestContext`, `buildContext` | Context of what? |
| `e`, `err` | `parseError`, `connectionError` | What error? |
| `i`, `j`, `k` | `routeIndex`, `retryAttempt` | Index of what? |
| `el`, `item` | `routeEntry`, `configOption` | Element of what? |
| `res`, `req` | `serverResponse`, `incomingRequest` | Spell it out |
| `opts`, `args` | `resolvedOptions`, `commandArguments` | Spell it out |
| `T`, `K`, `V` | `ElementType`, `KeyType`, `ValueType` | Describe the role |

### Naming rules

1. **Minimum two words** for locals and parameters -- unless a single complete word is unambiguous in context (e.g., `name`, `result`, `error`, `input`, `path`, `config`, `options`, `stream`, `value`, `schema`, `index`, `type`).
2. **Full words** -- never truncate. `config` is acceptable (universally understood). `arg` is not (use `argument`).
3. **Generic type parameters** use descriptive PascalCase:

```ts
type Registry<EntryType, KeyType extends string> = Map<KeyType, EntryType>;
```

4. **Loop variables** name what they iterate:

```ts
for (const route of routes) { ... }
for (let attemptIndex = 0; attemptIndex < maxRetries; attemptIndex++) { ... }
```

5. **Destructured fields** keep their original name -- no renaming to abbreviations:

```ts
const { statusCode, headers, body } = response;
```

---

## Type Conventions

### No `any`

Never use `any`. If you encounter `any` in existing code or find yourself reaching for it, stop and ask the user how they want to type it. There is almost always a concrete type, a generic, or `unknown` with a type guard that solves the problem properly. Leaving `any` in the codebase silently disables type checking and lets bugs through.

If a third-party library forces `any` into a signature, isolate it at the boundary and cast to a proper type immediately:

```ts
const rawResult = externalLib.parse(input);
const typedResult = rawResult as ParsedDocument;
```

### Readonly for API contracts

Use `readonly` on types that represent your module's public API surface -- configs passed in, results returned, and contracts consumers implement. This prevents accidental mutation of objects that cross module boundaries and protects framework/library contracts.

```ts
export interface RouterConfig {
  readonly name: string;
  readonly basePath: string;
  readonly middleware?: ReadonlyArray<Middleware>;
  readonly timeout?: number;
}
```

For internal working objects that are built up incrementally or mutated as part of their purpose (builders, accumulators, draft state), plain mutable types are appropriate:

```ts
interface RouteMatchBuilder {
  params: Record<string, string>;
  score: number;
}
```

### Discriminated unions

Use a `type` field as the discriminant:

```ts
export type ParseResult =
  | { type: "success"; value: Document }
  | { type: "error"; message: string; line: number }
  | { type: "partial"; value: Document; warnings: string[] };
```

### Resolved companion types

When a config has optional fields that become required after defaults are applied:

```ts
export interface CacheOptions {
  /** Time-to-live in ms. @default 60_000 */
  readonly ttl?: number;
  /** Max entries. @default 1000 */
  readonly maxSize?: number;
}

export interface ResolvedCacheOptions {
  readonly ttl: number;
  readonly maxSize: number;
}
```

The resolved type is internal -- consumers pass `CacheOptions`, the factory works with `ResolvedCacheOptions` after applying defaults.

---

## Minimal Data Transformation

Pass data structures through your code in their original shape whenever possible. Every transformation — mapping, reshaping, normalizing, converting to a DTO — adds a layer of indirection that obscures where data comes from and what shape it actually is. The more transformations between the source and the consumer, the harder it is to trace a bug or understand a data flow.

### Use types, not transformations

When you receive a data structure (from an API, a database, a config file, another module), type it and pass it along. Don't destructure it into a new object just to rename fields or drop properties you don't need right now. The downstream consumer can pick the properties it needs directly.

```ts
// Bad -- creates an intermediate shape that hides the original structure
function processUser(rawUser: RawUser) {
  const user = {
    id: rawUser.user_id,
    fullName: `${rawUser.first_name} ${rawUser.last_name}`,
    email: rawUser.email_address,
  };
  return calculatePermissions(user);
}

// Good -- pass the raw structure, type it, access what you need
function processUser(rawUser: RawUser) {
  return calculatePermissions(rawUser);
}

function calculatePermissions(user: RawUser) {
  if (user.role === "admin") { ... }
}
```

### When transformation is justified

Transform only when an algorithm or data structure requirement demands it:

- **Indexing** -- building a `Map` or lookup table from an array for O(1) access
- **Aggregation** -- reducing a collection to a summary value
- **Algorithm input** -- a sorting algorithm needs a comparator, a graph algorithm needs an adjacency list in a specific shape
- **Boundary crossing** -- converting between serialization formats (JSON to binary, protobuf to objects) at system edges

These are cases where the data *must* change shape for the computation to work. Renaming fields or extracting a subset "for cleanliness" is not algorithmic necessity -- it's cosmetic overhead that distances the code from its data source.

### The cost of unnecessary transformation

Each mapping step introduces:
- **A new type to maintain** -- the intermediate shape needs its own interface, which drifts from the source over time
- **Broken traceability** -- when a bug surfaces, you trace through multiple transformation layers instead of following the data directly
- **Wasted allocation** -- spreading, mapping, and reconstructing objects costs memory and CPU for no behavioral benefit

If a function only needs two fields from a large object, it takes the whole object and reads those two fields. The function signature documents what it depends on through its type, and the caller doesn't need to pre-select properties.

---

## Minimal Data Transformation

Pass data structures through your code in their original shape whenever possible. Every transformation — mapping, reshaping, normalizing, converting to a DTO — adds a layer of indirection that obscures where data comes from and what shape it actually is. The more transformations between the source and the consumer, the harder it is to trace a bug or understand a data flow.

### Use types, not transformations

When you receive a data structure (from an API, a database, a config file, another module), type it and pass it along. Don't destructure it into a new object just to rename fields or drop properties you don't need right now. The downstream consumer can pick the properties it needs directly.

```ts
// Bad -- creates an intermediate shape that hides the original structure
function processUser(rawUser: RawUser) {
  const user = {
    id: rawUser.user_id,
    fullName: `${rawUser.first_name} ${rawUser.last_name}`,
    email: rawUser.email_address,
  };
  return calculatePermissions(user);
}

// Good -- pass the raw structure, type it, access what you need
function processUser(rawUser: RawUser) {
  return calculatePermissions(rawUser);
}

function calculatePermissions(user: RawUser) {
  if (user.role === "admin") { ... }
}
```

### When transformation is justified

Transform only when an algorithm or data structure requirement demands it:

- **Indexing** -- building a `Map` or lookup table from an array for O(1) access
- **Aggregation** -- reducing a collection to a summary value
- **Algorithm input** -- a sorting algorithm needs a comparator, a graph algorithm needs an adjacency list in a specific shape
- **Boundary crossing** -- converting between serialization formats (JSON to binary, protobuf to objects) at system edges

These are cases where the data *must* change shape for the computation to work. Renaming fields or extracting a subset "for cleanliness" is not algorithmic necessity -- it's cosmetic overhead that distances the code from its data source.

### The cost of unnecessary transformation

Each mapping step introduces:
- **A new type to maintain** -- the intermediate shape needs its own interface, which drifts from the source over time
- **Broken traceability** -- when a bug surfaces, you trace through multiple transformation layers instead of following the data directly
- **Wasted allocation** -- spreading, mapping, and reconstructing objects costs memory and CPU for no behavioral benefit

If a function only needs two fields from a large object, it takes the whole object and reads those two fields. The function signature documents what it depends on through its type, and the caller doesn't need to pre-select properties.

---

## Comments and Documentation

### No file-level comments

Do not add comments at the top of files. The filename and its contents should be self-explanatory given the file organization conventions.

### Inline comments: less is more

Code should be readable through descriptive naming and clear structure. Only add inline comments when:

- **Complex conditionals** -- explain *why* a condition exists when the logic isn't obvious from the code alone
- **Data transformations** -- describe what shape the data is being converted to and why, especially when the transformation is algorithmic or non-trivial
- **Non-obvious business logic** -- explain the reasoning behind a specific approach when someone reading the code would reasonably ask "why?"

Do not comment on simple checks, bounds, or straightforward control flow. If you feel the need to comment basic logic, the variable and function names should be improved instead.

```ts
// Flatten nested route groups into a single lookup table keyed by full path,
// because the middleware chain resolves against complete paths not segments.
const flatRoutes = flattenRouteTree(routeGroups);
```

### No sub-section markers

Do not use `// -- Section name --` or similar decorative markers inside functions. They add noise. If a function is long enough to need section headers, it should be broken into smaller functions.

### No decorative separators

Lines like `// ===========================` or `// ---------------------` are prohibited.

### JSDoc

**Public API** (exported factories and interfaces) gets a JSDoc block with:
- 1-2 sentence description of what it does
- `@param` for each parameter
- One `@example` showing typical usage

```ts
/**
 * Create an HTTP router with pattern matching and middleware support.
 *
 * @param config - Router configuration including name, base path, and optional middleware.
 * @example
 * ```ts
 * const router = createRouter({ name: "api", basePath: "/v1" });
 * router.addRoute("/users/:id", handleGetUser);
 * ```
 */
export function createRouter(config: RouterConfig): Router {
```

**Interface fields** get single-line `/** */` with `@default` where applicable:

```ts
/** Base path prepended to all routes. */
readonly basePath: string;
/** Request timeout in milliseconds. @default 30000 */
readonly timeout?: number;
```

**Internal functions** (utils, helpers) get a brief 1-2 sentence JSDoc describing what they do. No `@param`, no `@example` -- the function signature and descriptive naming should make usage clear. Only add an example to an internal function if the usage is genuinely non-obvious.

```ts
/** Match a request path against registered route patterns, returning the first match. */
export function matchRoute(
  routes: Map<string, RouteHandler>,
  path: string,
): RouteMatch | undefined {
```

**Implementation-only API** uses `@internal`:

```ts
/** @internal Used by the plugin system -- not part of the public API. */
registerHook?(name: string, handler: unknown): void;
```

### Linter disables

Always include a reason:

```ts
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- third-party callback signature
```

---

## Test Philosophy

### Tests are behavioral specifications

Tests define *what should happen*, not what the code currently does. They are the source of truth for intended behavior.

**When a test fails:**

1. Do not modify the test to match the implementation. The test represents intended behavior.
2. Review the implementation to find where it diverges.
3. Fix the implementation to satisfy the test.
4. If the test expectation is genuinely wrong (misunderstood requirement), confirm with the user before changing it.

Write tests (or at least define expected behavior) before implementation when possible. The question is always "what *should* this return?", not "what *does* this return?"

### Test file organization

- **Co-located**: tests live next to the code they test as `{domain}.test.ts`
- **Shared fixtures** go in `test.utils.ts` (not `test-helpers.ts`, `helpers.ts`, or `mocks.ts`)
- **Test utilities are never exported from barrels**
- **Cross-package test imports** use the package name, not relative paths into another package's internals

### Test structure

```ts
describe("createRouter", () => {
  it("should match static routes exactly", () => { ... });
  it("should extract path parameters", () => { ... });

  describe("middleware", () => {
    it("should run middleware in order before the handler", () => { ... });
  });
});
```

- Top-level `describe` matches the function/module name
- Nested `describe` groups related scenarios
- `it` descriptions use `"should"` prefix

### Assertion style

- Inline literals over shared fixture objects
- `toEqual` for structural comparison of object shapes
- Track side effects with arrays (`const calls: string[] = []`)

---

## Detecting Project Context

When entering a project, look for these signals to adapt conventions:

| Signal | Convention to adapt |
|---|---|
| `bun.lockb` / `bunfig.toml` | Bun runtime -- use `bun:test`, Bun APIs |
| `vitest.config.*` | Vitest test runner |
| `jest.config.*` | Jest test runner |
| `deno.json` / `deno.lock` | Deno runtime -- use `Deno.test`, URL imports |
| `tsconfig.json` `strict: true` | Match strictness level |
| Existing `readonly` usage | Match project's immutability conventions |
| Existing class-based code | Note it, but use factories for new domain code unless the project is consistently class-based |
| `.eslintrc` / `biome.json` | Follow the project's linter rules |

When in doubt, apply the conventions in this skill. When conventions conflict with an established project pattern, follow the project and note why.

---

## Keeping Documentation in Sync

Before making code changes, check whether the project has documentation (a `docs/` directory, README files, API references, markdown guides, etc.). When it does, any code change that alters public-facing behavior must include corresponding documentation updates.

### What triggers a docs update

- **Renamed or removed exports** -- update any docs that reference the old name
- **Changed function signatures** -- update parameter descriptions, usage examples
- **Changed return types or behavior** -- update descriptions of what the function does
- **New configuration options** -- add them to the relevant docs page

### Keep updates minimal

Match the scope of the docs change to the scope of the code change. If you renamed a parameter, update the line that references it -- don't rewrite the entire page. Documentation churn makes it harder to review what actually changed.

The exception is implementing a new feature. New features warrant a dedicated documentation section or page that explains what it does, when to use it, and shows a usage example.

### Don't create docs unprompted

If the project has no existing documentation, do not create any. Only update what already exists. If a new feature clearly needs docs and none exist, ask the user whether they want documentation added and where.
