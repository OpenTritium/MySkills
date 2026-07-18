---
name: rust-structure-refactor
description: "Guide broad Rust structural changes: inline or decompose functions, split cohesive structs, and set module API visibility. Use when the question is a function, struct, or module boundary, ownership, or export; use rust-method-placement for the home of one operation, func-smell for one-function contract smells, and testing-strategy when behavior must be characterized before edits. 中文触发：结构重构、函数拆解、短函数内联、struct 拆分、模块导出、可见性控制、pub(crate) 滥用。"
---

# Rust Structure Refactor

## Overview

Use this skill to refactor Rust structure while preserving behavior and making ownership, responsibilities, and API boundaries explicit. Focus on whether a symbol deserves to exist, where it should live, and which visibility should expose it; defer naming-only issues to `naming-smell`, function contract smells to `func-smell`, and type invariants to `encode-invariant`.

## Refactoring Workflow

1. **Map the current boundary** — Preserve behavior by inspecting the module tree, public re-exports, callers, fields, mutation points, error propagation, and tests. Mark orchestration, pure logic, I/O, validation, state transition, and formatting.

2. **Draw the dependency direction** — For each candidate extraction, list inputs, outputs, borrowed data, owned data, and side effects. Prefer return values over output parameters and pass the smallest cohesive value rather than the entire `self` or a catch-all context.

3. **Choose the smallest structural change** — Inline a needless wrapper, extract a cohesive phase, introduce a value object, split a state enum, or move a responsibility to its owning module. Do not perform all transformations at once.

4. **Apply visibility at the boundary** — Start with private items and add only the visibility needed by the parent module or exported API. Centralize external exposure in `lib.rs`, `main.rs`, or the parent module's `pub use` statements.

5. **Validate after each seam** — Run formatting, compile checks, focused tests, and then broader tests as appropriate. Compare behavior, public API, error types, and performance-sensitive paths before and after. If behavior or a bug contract is uncertain, follow **Behavior Before Structure**.

## Behavior Before Structure

Do not refactor an uncertain behavior from intuition alone:

- Write focused tests through the stable boundary before changing production structure.
- Cover the relevant happy path and unhappy path; add edge cases only when they express the contract.
- Characterize ambiguous legacy behavior first. For a confirmed bug, make the desired regression test fail before fixing it.
- Refactor only after the tests make the behavior explicit, then keep them as the contract guard.

## Short Function Decisions

Judge source-level inlining separately from the compiler's `#[inline]` hint. Do not add `#[inline]` merely because a function is short; let optimization and profiling justify that attribute.

| Candidate | Action |
|---|---|
| One-use helper that only forwards arguments or returns one expression | Inline it if the call site remains readable |
| One-use helper with a domain name, invariant, error policy, or meaningful phase | Keep it; the name is carrying design information |
| Reused, recursive, independently tested, or independently mocked logic | Keep or extract it as a real unit |
| Tiny helper that only compensates for a poor call-site shape | Fix the data or API shape before deciding |
| Helper whose only reason for existence is a long parent function | Inspect its responsibility and dependencies; do not extract blindly |

Inline only when the resulting expression or block still has a clear purpose and does not duplicate a non-trivial operation. Keep a short helper when it protects a domain term, centralizes a rule, hides unsafe or delicate code, or creates a stable boundary likely to be tested or reused.

## Decompose Long Functions

Use the function's data flow and responsibility changes to choose seams:

1. **Name the phases** — For example: load, parse, validate, transform, persist, and notify. If the top-level sequence cannot be named, understand the behavior before extracting.
2. **Extract pure logic first** — Move calculations and transformations with explicit inputs and outputs. These are easiest to test and least likely to change ownership semantics.
3. **Extract boundary adapters next** — Isolate database, filesystem, network, logging, and serialization operations. Make their failure and side effects visible in the signature or name.
4. **Leave an orchestrator** — The original function should coordinate meaningful steps and preserve ordering, early returns, error mapping, and cleanup. It should not become a parameter-passing script with dozens of helpers.
5. **Move code with its data** — If a block needs a stable subset of fields, consider a cohesive struct or a method on the type that owns the invariant. Avoid passing `&mut self` when a smaller input and returned value suffice.

For async code, do not move a block across an `.await` without checking borrow lifetimes, `Send` requirements, cancellation behavior, held locks, and resource cleanup. For fallible code, preserve the exact error boundary and context instead of replacing meaningful errors with generic ones.

## Struct Decomposition

Split a struct when a field group has a domain name, its own invariant or lifecycle, a distinct owner, independent change pressure, or a useful unit of construction. Keep fields together when callers always read and write them together under the same invariant.

Use composition and move behavior with the data it protects:

```rust
struct Order {
    identity: OrderIdentity,
    pricing: Pricing,
    fulfillment: Fulfillment,
}

struct Pricing {
    subtotal: Money,
    discount: Money,
    total: Money,
}
```

Then give each component the smallest methods needed to maintain its own rules. Keep fields private and expose operations or validated constructors instead of a bag of getters and setters.

Prefer an enum when a large struct encodes mutually exclusive states with `Option` fields or booleans. Prefer a newtype when a field or parameter has a domain identity, validation rule, or argument-swap risk; use `encode-invariant` for the type-level details.

Do not split merely because a struct has many fields. Avoid one-field wrappers with no invariant, `*Data`/`*Context` dumping grounds, duplicated copies of the same state, and structs whose methods still operate on unrelated fields. When splitting, update constructors, methods, tests, serialization, and ownership together.

## Visibility And Module Exports

Use the narrowest visibility that expresses the dependency:

| Visibility | Meaning | Default use |
|---|---|---|
| private | Usable only in the defining module and its descendants | Implementation details |
| `pub(super)` | Usable by the immediate parent module | Child-to-parent collaboration |
| `pub(in path)` | Usable within one named ancestor path | Rare, precise internal boundary |
| `pub(crate)` | Usable by every module in the crate | Exception for a genuinely crate-wide internal contract |
| `pub` | Reachable outside the crate when the path is public | Deliberate external API |

Prefer a single export boundary. For example, let `lib.rs` declare `mod orders;` and expose `pub use orders::{Order, create_order};`. The `orders` module can keep helpers private and expose only the symbols that its parent intentionally re-exports.

Use `pub(crate)` only after identifying multiple sibling consumers and documenting why `pub(super)` or a private re-export cannot express the relationship. Do not add `pub(crate)` to make a privacy error disappear, and do not make every leaf module `pub`.

When moving code, check whether the public API can remain unchanged. If an internal symbol must cross a module boundary, first ask whether the parent should own the operation, whether a `pub use` re-export is enough, or whether the data belongs in a shared domain type. Keep internal paths out of downstream callers.

## Compact Example

Before, a crate-wide entry point often hides the intended boundary:

```rust
pub(crate) fn process_order(input: RawOrder) -> Result<Order, Error> {
    let parsed = parse(input)?;
    let order = validate(parsed)?;
    persist(&order)?;
    Ok(order)
}
```

After, keep the module private and export one stable operation from the parent:

```rust
mod orders;
pub use orders::create_order;
```

Keep `create_order` as the orchestration boundary and keep `parse`, `validate`, and `persist` private in `orders`. Inline any one-use helper that adds no phase name, invariant, or independent boundary.

## Review Checklist

- Does each remaining helper have reuse, semantic meaning, a test seam, or a real boundary?
- Does the long function read as one level of named phases after extraction?
- Do extracted functions accept focused inputs and return values without output arguments?
- Does each struct have coherent fields, invariants, lifecycle, and behavior?
- Are fields private and public symbols exported from one intentional module boundary?
- Is every `pub(crate)` justified by a true crate-wide internal contract?
- Are errors, ordering, locks, transactions, async cancellation, and cleanup unchanged?
- If behavior was uncertain, did happy/unhappy characterization or regression tests precede the structural change?
- Did focused tests and compile checks run after each structural seam?

## Reporting Format

When reviewing rather than editing, report the smell and evidence, the smallest safe transformation, the before/after visibility and signature plan, and validation plus residual risks.
