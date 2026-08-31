---
name: rust-practices
description: Rust implementation and review conventions for architecture-first libraries and applications. Use this skill whenever writing, modifying, refactoring, or reviewing Rust code; designing Rust modules, public APIs, errors, traits, ownership, or concurrency; adding Cargo dependencies; or discussing Rust best practices. Apply these conventions even when the request only mentions a Rust file, crate, Cargo workspace, `mod.rs`, `Result`, `anyhow`, `thiserror`, traits, or ownership.
---

## Purpose and Precedence

Build Rust systems as strongly encapsulated, hierarchical, domain-oriented architectures. The filesystem, module hierarchy, visibility, types, ownership, and public exports should describe the same conceptual system.

Apply this skill to implementations and reviews. Its conventions take precedence over local Rust style unless a concrete compatibility, framework, generated-code, or public-API constraint prevents it. Explain such a divergence briefly in the final response.

Architecture is not a substitute for correctness. Changes must compile, preserve intended behavior, protect invariants, and receive appropriate tests.

Prioritize decisions in this order:

1. Correct domain semantics and behavior.
2. Clear ownership and dependency direction.
3. Encapsulation, invariants, and strong type modeling.
4. A simple concrete implementation.
5. Narrow visibility and a deliberate public API.
6. Testability and documented behavior.
7. Extensibility, performance, and concurrency only when supported by a current requirement or evidence.

## Naming, Documentation, and Comments

Use complete, descriptive names. A reader should understand a variable's role without decoding an abbreviation or relying on nearby context.

```rust
// Avoid
for i in 0..records.len() {
    let cfg = &configs[i];
}

// Prefer
for record_index in 0..records.len() {
    let record_configuration = &record_configurations[record_index];
}
```

NEVER use shorthand, unexplained abbreviations, or single-letter names for meaningful values. Prefer semantic names such as `request_identifier`, `record_index`, `source_path`, and `retry_attempt`.

Keep implementation in-line code comments minimal. Write a comment only when it explains a non-obvious invariant, tradeoff, external constraint, or a complex algorithm whose loops and conditionals would otherwise obscure intent. Do not narrate straightforward code.

Document every public API with rustdoc, including public structs, enums, traits, functions, type aliases, and public fields. Describe what it represents or does, meaningful failure behavior, and observable side effects. Provide a concise runnable example for public behavior that is non-obvious or likely to be used directly.

Document every struct field, including non-public fields when the field's role is not self-evident. Keep field descriptions focused on the meaning and invariant of the stored value, not its Rust type.

When a function accepts `&mut T`, document which parameter is mutated and the resulting state or invariant. Prefer narrower mutable inputs over passing a broad parent system just to reach a field.

```rust
/// Sorts `records` in place by their creation time.
pub fn sort_records(records: &mut [Record]) {
    records.sort_by_key(|record| record.created_at);
}
```

## Module Architecture

Build domain modules from the following template. Substitute the singular, snake_case domain name for `{module_name}`. A folder is a module and always has `mod.rs`; do not create one-off operation files directly under another module.

```text
{module_name}/
├── mod.rs                 // Declares children and selectively re-exports the module API.
├── {module_name}.rs       // Defines the module's primary struct and its inherent behavior.
├── types.rs               // Small supporting structs or aliases that belong to this module.
├── utils.rs               // Pure helpers for this module; no domain orchestration or I/O.
├── constants.rs           // Constants shared within this module.
├── errors.rs              // Error types for this module's own behavior only.
├── tests.rs               // The only white-box test file; contains inline test modules per source file.
├── identifiers.rs         // Use only for one or two small identifier newtypes.
├── enums.rs               // Use only for one or two small, data-only enums.
├── identifiers/           // Replace identifiers.rs when identifiers need separate behavior.
│   ├── mod.rs
│   ├── errors.rs          // Validation and construction errors for identifiers only.
│   ├── {first}_identifier.rs
│   └── {second}_identifier.rs
├── enums/                 // Replace enums.rs when enum types have independent behavior.
│   ├── mod.rs
│   ├── {first}_kind.rs
│   └── {second}_status.rs
└── {operation_name}/      // A cohesive operation owned by this domain.
    ├── mod.rs             // Declares children and selectively re-exports operation API.
    ├── {operation_name}.rs
    ├── errors.rs          // Errors produced by this operation only.
    └── tests.rs
```

For every module that contains implementation behavior, create and declare `{module_name}.rs` and `tests.rs`; write at least one focused white-box test when the module has private behavior or invariants. `tests.rs` is a single file. It contains one outer inline `mod tests` and one inline child module per relevant sibling source file. Do not create `tests/`, `test/`, or any other test directory beneath `src/`; the crate-root `tests/` directory is reserved for black-box integration tests. Create and declare `errors.rs` whenever that module can reject input, perform fallible work, or translate a lower-level failure. This includes application operations that ultimately return `anyhow::Result`: keep operation-specific failure classification or parsing errors in the operation's `errors.rs`, then add `anyhow` context at the application boundary. `types.rs`, `utils.rs`, `constants.rs`, `identifiers.rs`, and `enums.rs` are optional and must not be empty. `identifiers.rs` and `enums.rs` are the two deliberate exceptions to the directory rule. Promote either to its shown directory form when it defines more than two types, a type needs substantial inherent or trait behavior, or it needs dedicated errors or tests. Do not split a small group merely to fill the template.

The primary struct's name and file must match exactly: `Entity` is in `entity/entity.rs`, `Task` is in `task/task.rs`, and `Board` is in `board/board.rs`. Do not call a core entity `EntitySummary` when it represents the domain `Entity`; reserve summary, view, response, and snapshot names for genuinely reduced or derived representations. Do not put a primary entity and a distinct operation abstraction in the same source file.

Build a hierarchy before writing implementation files. First identify the bounded domain that owns the language, invariants, and lifecycle. Then place its entity, operations, validation, persistence boundary, and domain-specific transport beneath that domain. Finally, use child modules to encapsulate a coherent local responsibility. Prefer this domain flow over a flat list of crate-root modules or parallel technical trees, because a change to one concept should lead a reader down one directory tree rather than across the whole crate.

Treat a bounded domain as a valid parent even when it has no aggregate struct of its own. The parent earns its place by owning the concept and its relationships, not only by storing fields. Do not flatten `entity`, `entity_importer`, `entity_validator`, and `entity_repository` into sibling crate-root modules merely because no `Entities` struct exists. If those capabilities create, validate, retrieve, or otherwise operate on the same entity lifecycle, they belong beneath `entity/`.

Use this ownership test before choosing a location:

```text
Does the behavior enforce, create, transform, retrieve, validate, or expose one domain's data?
  Yes -> place it in a descriptive child module of that domain.
  No  -> Does it coordinate multiple domains or provide neutral shared vocabulary/infrastructure?
           Yes -> give it a deliberate crate-root boundary.
           No  -> identify the missing domain owner before adding the module.
```

The resulting domain hierarchy should look like this:

```text
src/
├── lib.rs
├── entity/                    // The domain, and everything that is about it.
│   ├── mod.rs                 // The domain boundary; re-exports the whole entity API.
│   ├── entity.rs              // The primary struct.
│   ├── identifiers.rs
│   ├── errors.rs
│   ├── tests.rs
│   ├── repository/            // What storage must provide, in this domain's types.
│   │   ├── mod.rs
│   │   ├── entity_repository.rs
│   │   └── errors.rs
│   ├── service/               // The use cases performed on this domain.
│   │   ├── mod.rs
│   │   ├── entity_service.rs
│   │   ├── errors.rs
│   │   └── tests.rs
│   ├── routes/                // The transport that exposes this domain.
│       ├── mod.rs
│       ├── entity_routes.rs
│       ├── types.rs
│       └── errors.rs
│   └── lookup/                // A local operation over Entity data.
│       ├── mod.rs
│       ├── entity_lookup.rs
│       ├── errors.rs
│       └── tests.rs
└── event_bus/                 // Neutral infrastructure used by several domains.
    ├── mod.rs
    └── event_bus.rs
```

Name a domain's children for the role they play -- `repository`, `service`, `routes`, `importer` -- and keep the primary file named after the primary type it defines, so `EntityService` is in `entity/service/entity_service.rs`. The role is the module's name; the type keeps its qualified name. Re-export through the domain's `mod.rs` so consumers write `crate::entity::EntityService` rather than `crate::entity::service::entity_service::EntityService`.

A parent module must still mean something. Group by a domain, a bounded concept, or a subsystem that owns state, invariants, behavior, configuration, a lifecycle, or a public aggregate API. Do not create a parent whose only content is a category label -- `helpers`, `common`, `shared`, `models`, `impls`, `misc` -- or one that groups modules solely because they use the same technology rather than because they serve the same concept. Crate-root modules are the exception: reserve them for neutral shared vocabulary such as `money`, cross-cutting infrastructure such as `event_bus`, and explicit multi-domain orchestration. A domain-specific operation is not a crate-root peer by default.

Cross-domain code belongs where the dependency direction stays legal. A type two domains both depend on moves to a crate-root module they may both reach, never into one of them. A transport that spans several domains -- a router that assembles them, a subscription stream that forwards all of their events -- is its own crate-root module rather than a child of whichever domain it mentions first.

An importer that creates or validates one domain's records belongs under that domain: `src/entity/importer/mod.rs`, `src/entity/importer/entity_importer.rs`, `src/entity/importer/errors.rs`, and `src/entity/importer/tests.rs`, not a one-off `src/importer.rs`. Make an importer a crate-root module only when it intentionally coordinates unrelated domains. A board is a legitimate root module because it owns board state and behavior: `board/mod.rs` is the board boundary, `board/board.rs` defines `Board` and exposes board behavior, and task is a controlled child at `board/task/mod.rs` with `board/task/task.rs` and `board/task/tests.rs`. Keep task construction and representation private unless callers need them to use the board API.

Each `mod.rs` declares private child modules, re-exports only the intended interface, documents the subsystem, and declares its local `tests.rs`:

```rust
mod errors;
mod lookup;

#[cfg(test)]
mod tests;

pub use errors::EntityLookupError;
pub use lookup::EntityLookup;
```

The required `{module_name}/mod.rs` plus `{module_name}/{module_name}.rs` layout intentionally triggers Clippy's `module_inception` lint. Place `#![allow(clippy::module_inception)]` at the top of that specific `mod.rs`, with a short comment stating that the allow preserves this repository's explicit module template. Do not weaken Clippy globally or add broad lint allowances.

`tests.rs` mirrors the module's immediate implementation files with inline test modules so white-box tests stay local. It must remain one file and use this shape. Include a child module only when its matching source file exists. When `identifiers/` or `enums/` is a directory, its own `tests.rs` owns those tests instead of the parent test file:

```rust
mod tests {
    mod entity {
        use super::super::entity::Entity;

        #[test]
        fn constructs_a_valid_entity() {
            // Exercise Entity's private invariants here.
        }
    }

    mod utils {
        // Tests for utils.rs only.
    }

    mod identifiers {
        // Include only when identifiers.rs is a single file.
    }

    mod errors {
        // Tests for errors.rs only.
    }

    mod enums {
        // Include only when enums.rs is a single file.
    }
}
```

Keep implementation modules private unless the module itself is intentionally public. Consumers use selectively re-exported types such as `crate::entity::EntityLookup`, never implementation paths such as `crate::entity::lookup::entity_lookup::EntityLookup`. Put black-box public API tests under the crate-level `tests/` directory.

Reserve `utils.rs` for pure helper functions that operate on the containing module's data but do not belong naturally on the primary type. Put validation owned by identifier construction in `identifiers.rs` or `identifiers/`, not in a general utility file. Do not use `utils.rs` as a miscellaneous dump.

## Dependencies and Visibility

Treat dependency direction as an architectural invariant:

```text
Domain -> shared domain type      preferred: identifiers, errors, and operations may use the entity type they serve.
Parent(Domain) -> child                 preferred: an aggregate composes and orchestrates its owned modules.
Sibling -> sibling              preferred within one domain: `service` may depend on `repository`, and `routes` on `service`.
Sibling -> sibling              deliberate only across domains: depend on the other domain's re-exported API, never its internals.
Child -> parent                 allowed for the domain's own type: a child may use the entity its parent defines.
Child -> parent                 prohibited otherwise: do not pass the aggregate into an owned child so it can reach data indirectly.
```

An operation such as `EntityLookup` may consume `Entity` because `Entity` is the shared domain type it serves. It must not depend on a synthetic aggregate just to obtain that type or its data. Pass the smallest semantic dependency, such as `&EntityRepository` or `&[Entity]`, rather than a large parent system. A deep upward-reaching import, a repeated dependency between two domains, or visibility widening is an architectural signal: move the shared type to a real common owner, pass a narrow capability, or remove an unnecessary wrapper module.

Grouping a domain does not flatten its layers. Inside `entity/`, the arrow still runs `routes -> service -> repository -> entity`: the entity knows nothing of its repository, the repository nothing of its service, and the service nothing of its routes. Domain grouping decides where a module lives; it never licenses a lower layer to reach up into a higher one.

Use the narrowest visibility that expresses the intended permission:

```text
private -> pub(super) -> pub(in crate::module) -> pub(crate) -> pub
```

Do not widen visibility solely to satisfy the compiler. Before widening it, ask whether the consumer should know about the item, whether behavior should cross the boundary instead of state, and whether a selective re-export is more accurate.

Keep fields private by default. A subsystem owns its invariants and should expose behavior rather than mutable representation. Every meaningful mutable state has one identifiable authoritative owner; model snapshots and derived state explicitly so mutable copies cannot silently diverge.

## Domain Modeling and APIs

Use semantic domain types instead of primitives when equal Rust types have different meanings. Prefer newtypes and enums over boolean flags.

```rust
// Avoid: two unrelated strings and an unexplained flag.
fn load_entity(entity: String, owner: String, include_inactive: bool) {}

// Prefer: the type system documents valid inputs.
fn load_entity(entity_identifier: EntityIdentifier, owner_identifier: OwnerIdentifier, visibility: EntityVisibility) {}
```

Make invalid states difficult to construct through private fields, validated constructors, meaningful enums, dedicated configuration types, and typestate only when its extra complexity protects an important transition.

Use `new` for infallible construction and `try_new` or `TryFrom` when construction validates input. Use a builder when construction has several independently optional settings. Do not expose public fields that permit callers to bypass invariants.

Implement standard traits when their ordinary semantics apply, such as `Debug`, `Clone`, `Eq`, `Ord`, `Hash`, `Display`, `From`, `TryFrom`, `AsRef`, and `IntoIterator`. Prefer implementing `From` and `TryFrom` rather than their reciprocal conversion traits. Do not derive or implement a trait merely because it is possible.

Add `#[must_use]` to public values whose ignored result usually indicates a mistake, including fallible or state-transforming operations where that contract is meaningful.

Keep generic bounds on the methods or implementations that need them, not on a generic type declaration that can otherwise remain unconstrained.

## Abstractions and Dispatch

Start concrete. Introduce a trait only for a real capability, multiple implementations, substitution, generic algorithm, dependency inversion, runtime-selected implementation, or external extension point. Do not imitate interface-per-class architectures.

Define a capability trait near the consumer when only that consumer needs it; define a fundamental domain capability with the domain abstraction. Keep traits small and behavior-oriented.

Use an enum for a closed set of alternatives and a trait for an intentionally open behavioral capability. Prefer static dispatch with generic bounds or `impl Trait` unless runtime heterogeneity is an actual requirement. Use `dyn Trait` deliberately and account for object safety.

Use associated types when an implementation determines one canonical related type; use generic trait parameters when one implementation has multiple meaningful relationships. Model the relationship rather than choosing mechanically.

## Errors and Resources

In applications, use `anyhow` for error propagation and contextual reporting at orchestration or boundary layers. In libraries, expose typed, domain-specific errors, normally from the subsystem's `errors.rs`, so callers can handle meaningful failure modes.

Child errors remain specific to child subsystems. A parent may compose or translate them at an architectural boundary while preserving useful sources and context. Avoid one global error type containing every application failure.

Do not use production `unwrap` or `expect` for recoverable failure. Use them only when a documented, local invariant makes failure unrecoverable; make the invariant clear in the message or nearby documentation. Propagate, handle, or map expected failures instead.

Use RAII to own resources such as files, sockets, transactions, and locks. Bound untrusted input, buffering, retry attempts, and I/O waits according to the domain. Prefer streaming over unbounded collection when input can be large.

## Ownership and Concurrency

Establish ownership and mutation boundaries before choosing concurrency machinery. Do not introduce `Arc`, locks, atomics, channels, or interior mutability because concurrency may exist later.

Before adding synchronization, identify the authoritative owner, writers, readers, transfer or lifetime requirements, actual concurrent access, and required synchronization semantics. Then choose the smallest correct tool: ordinary ownership or references, ownership transfer, immutable `Arc<T>`, message passing, a lock, or atomics.

Do not default shared state to `Arc<Mutex<T>>`. Concurrency primitives solve a specific access requirement; they do not define an architecture. This skill covers synchronous libraries and applications; ask before introducing async-runtime-specific patterns.

## Dependencies, Compatibility, and Performance

Prefer the standard library and dependencies already present in the workspace. Add a crate only when it provides clear value that outweighs maintenance, security, compile-time, and dependency-tree cost. Enable only necessary features and avoid default features when they pull in unused capabilities.

For public libraries, treat public types, trait bounds, error variants, feature flags, and re-exports as compatibility commitments. Do not make accidental breaking changes. Keep public APIs smaller than implementation detail, but expose the domain surface required by intended consumers.

Optimize based on evidence. First select a clear algorithm and ownership model; then measure a demonstrated hot path before adding complexity. Review allocation, cloning, collection choice, and asymptotic behavior when profiling or requirements identify them as material.

## Testing and Validation

Tests specify observable behavior and protect domain invariants. Cover normal behavior, meaningful boundaries, failure paths, and regressions. Test public APIs through integration tests; use subsystem tests for private behavior that cannot be exercised adequately through the public interface.

Keep tests deterministic. Control time, randomness, external I/O, and environment through real domain dependencies when production behavior requires them, not through test-only API parameters. Prefer focused fakes over broad mocks when a dependency must be isolated.

When practical, validate Rust changes with the affected crate or workspace:

```text
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Use repository-documented feature, workspace, platform, or CI commands when they are stricter or more representative. Report checks that could not be run and why.

## Implementation and Review Checklist

For each non-trivial change, review these questions before finishing:

1. Is the concept under its narrowest semantic owner and represented with the required `mod.rs` hierarchy?
2. Does every new import point in an intended architectural direction?
3. Is the dependency the smallest semantic type or capability required?
4. Are visibility and re-exports intentional, with implementation details kept private?
5. Does one owner control each mutable state and its invariants?
6. Do domain types prevent ambiguous primitives, boolean flags, or invalid construction where it matters?
7. Is a concrete type sufficient, or is a trait, generic, dynamic dispatch, or concurrency primitive justified today?
8. Do errors match the application-versus-library policy and avoid unjustified panics?
9. Are names descriptive, public APIs and fields documented, mutation contracts stated, and comments limited to non-obvious reasoning?
10. Is any new dependency justified and minimally featured?
11. Do tests cover behavior, boundary conditions, failures, and regressions appropriate to the change?
12. Have formatter, Clippy, and tests been run or their absence reported?
