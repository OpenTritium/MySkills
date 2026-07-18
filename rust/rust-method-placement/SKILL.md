---
name: rust-method-placement
description: "Choose the owner and API form of a Rust operation or extracted helper: inherent or associated method, trait or extension trait, domain newtype, or free function. Use when a candidate behavior needs placement after helper extraction, API review, primitive-domain analysis, or foreign-type extension. Use rust-structure-refactor for broad function/struct/module decomposition, func-smell for one-function contract smells, and rust-api-consolidation for merging or removing existing APIs. 中文触发：方法归属、函数还是方法、自由函数、关联方法、扩展方法、扩展 trait、newtype、新增小函数、AI 重构新增小函数。"
---

# Rust Method Placement

## Core Rule

Place behavior with the type or module that owns its meaning, state, and invariant. Do not force every helper into a method: use a newtype when a domain concept needs an owner, and use a free function when no type is semantically primary.

## Workflow

1. **Map the candidate** — Record its inputs, outputs, state access, side effects, errors, validation, callers, and whether it is a one-use extraction or reusable operation.
2. **Find the owner** — Identify the type whose rule or state must change with the behavior. Passing `self` conveniently does not establish ownership.
3. **Check the boundary** — Inspect module ownership, foreign types, public exports, cross-component coordination, and visibility before adding a symbol.
4. **Choose and validate** — Apply the table below, then verify caller readability, invariant ownership, error and side-effect contracts, and focused tests.

## Decision Table

| Evidence | Prefer | Reason |
|---|---|---|
| Reads or changes one type's state, or maintains its invariant | Inherent method | The receiver owns the state transition |
| Constructs, parses, or validates a type without an instance | Associated function such as `new`, `from_parts`, or `try_from` | The type owns the construction contract |
| One capability has the same contract for multiple implementors | Trait method | Polymorphism is a real requirement |
| Behavior belongs ergonomically on a foreign or unmodifiable type, and method syntax materially helps | Extension trait | A local trait adds the method without changing the target type |
| A primitive, string, tuple, or raw identifier has domain identity, validation, or swap risk | Validating newtype with methods | The wrapper owns the concept and prevents misuse |
| Inputs have equal semantic status or no type is the natural owner | Free function | A false receiver would misrepresent the relationship |
| Coordinates I/O or several components | Module-level function or service object | The workflow boundary owns the coordination |
| Generic algorithm over borrowed inputs has no domain owner | Free function or appropriate trait | Generic reuse should not invent ownership |

Do not create a trait solely to turn a one-implementation free function into a method. For an extension trait, keep the trait local, import scope deliberate, and method names collision-safe. Do not use it to hide a generic utility, add a trivial one-use helper, or replace a newtype that should enforce an invariant.

## Small Helper Rule

- Inline a one-use helper that only forwards arguments, unwraps a field, or returns one uncomplicated expression when the call site stays clear.
- Keep a short method when its name carries a domain operation, invariant, state transition, delicate boundary, or meaningful test seam.
- Use an associated function for construction or validation, not behavior on an existing instance.
- Use an extension trait for method-like behavior on a type that cannot receive an inherent method; check trait scope and collisions first.
- Use a newtype when a repeated domain primitive needs validation, conversions, or stable behavior ownership.
- Use a free function when no receiver is primary, the operation is symmetric, or it coordinates unrelated owners.
- Do not use `&mut self` as a dumping ground. Move cohesive field groups into a component type or pass focused inputs and return the result.

The objective is one obvious home for the next rule change, not the maximum number of methods.

## Newtype Test

Introduce a newtype only when it has a distinct domain identity, a meaningful invariant or misuse risk, and at least one operation or boundary behavior to own. Otherwise prefer the existing type or a free function. Consult `encode-invariant` for representation, serde, boxed storage, and invalid-state details.

## Free Function Boundary

Prefer a free function when no receiver is primary, inputs are symmetric, unrelated types are being transformed, multiple services are coordinated, or the module—not a data type—owns the public workflow. Make dependencies explicit in its signature and keep it private unless callers need the module-level operation.

## Review Checklist

- Is ownership based on the rule or state rather than parameter convenience?
- Does a method use a cohesive receiver and preserve its invariant?
- Is an associated function about construction or validation?
- Is an extension trait justified by an unmodifiable target and meaningful method syntax?
- Does a newtype prevent real misuse or own real domain behavior?
- Does a free function honestly have no natural receiver or coordinate multiple owners?
- Does the change avoid one-use wrappers and needless public API growth?
- Are errors, side effects, ownership, async boundaries, and tests unchanged?

## Related Skills And Boundaries

- Use `rust-structure-refactor` for broader function, struct, and module decomposition; use this skill for the home of a particular operation.
- Use `encode-invariant` for type-level newtype design and invariant encoding.
- Use `func-smell` for parameter explosion, hidden effects, mixed abstraction levels, and other function-shape problems.
- Use `rust-api-consolidation` for merging, removing, or preserving existing APIs.
- Use `architecture-entropy-review` when a refactor creates duplicate owners, parallel routes, or cross-module drift.

## Reporting Format

When reviewing rather than editing, report the candidate, ownership evidence, selected form, rejected alternatives, smallest safe change, and focused validation. Explain why a free function has no natural owner or which invariant a method/newtype centralizes.
