---
name: encode-invariant
description: 'Use when reviewing Rust types for post-construction immutability or domain invariants, including the integration cost of boxed types and alias-versus-newtype choices. Prefer boxed slices/strings, shared unsized forms, NonZero types, newtypes, and enums that make invalid states unrepresentable. Keywords: boxed slice, into_boxed_slice, boxed str, into_boxed_str, invariant, immutable, Vec-to-boxed-slice, String-to-boxed-str, serde Deserialize, builder macro, type alias, newtype, NonZero, type state, CompactString, smallvec, niche optimization, 类型不变, 盒装切片, 新类型'
---

# Encode Invariant

## Core Question

Does the representation make the invariant explicit without imposing disproportionate construction, API, or interoperability cost?

## Rules Engine

1. **Freeze with Box** — `Vec<T>` → `Box<[T]>`, `String` → `Box<str>` when built once then read-only. `.into_boxed_slice()` / `.into_boxed_str()` at the finish. This removes resize operations and may reduce metadata on common 64-bit layouts; verify ownership and layout tradeoffs.

   **Integration friction matters.** Check `Deserialize` and builder paths before converting: deserialization into an owned string followed by boxing may add allocation and conversion work. Keep the original type when that friction outweighs the guarantee; otherwise centralize conversion at one boundary.

   **Container versus contents.** `Box<[T]>` freezes length, not `T`; mutable slice or element access remains possible. `Arc<[T]>` adds shared ownership and normally limits access to shared borrows; it is not deep immutability.

2. **Share without Fat** — `Arc<Vec<T>>` = 3 layers (Arc → Vec → heap); `Arc<[T]>` = 2. Same for `Rc<String>` → `Rc<str>`. Convert at construction: `Arc::from(vec.into_boxed_slice())`.

3. **Choose alias versus newtype deliberately** — Use a newtype for boundary types, validation, domain-specific methods, or argument-swap protection. Prefer a type alias when the type is documentation-only and arithmetic operators, comparisons across many sites, or serialization interop with external libraries are central. An alias buys ergonomics but provides no type protection; a newtype buys enforcement at the cost of conversions and trait forwarding.

4. **Prove Constraints at the Boundary** — Use `NonZero` or a validating newtype for domain constraints. Validate once at construction; consumers are safe by construction.

5. **Enum > (bool + Option)** — Replace correlated flags and optional fields with enum states so impossible states cannot compile.

6. **Niche Matters** — `Option<Box<T>>` = same size as `Box<T>` (null is niche). `Option<NonZeroU32>` = same size as `u32` (zero is niche). For optional values with an absent sentinel, `Option` is free.

## Transformation Table

| From | To | Savings |
|---|---|---|
| `Vec<T>` (read-only after build) | `Box<[T]>` | Removes capacity mutation; may reduce metadata |
| `String` (read-only after build) | `Box<str>` / `Arc<str>` | Removes capacity mutation; may reduce metadata |
| `Arc<Vec<T>>` | `Arc<[T]>` | Fewer layers; layout benefit is platform-dependent |
| `Rc<String>` | `Rc<str>` / `Arc<str>` | Fewer layers; layout benefit is platform-dependent |
| Boundary values with swap or validation risk | Newtype | distinct signatures and invariant enforcement |
| `u64` (documentation only, arithmetic-heavy) | Type alias of `u64` | operator and serialization interop; no type protection |
| Values with a non-zero or domain constraint | `NonZero` / validating newtype | compile-time guarantee after construction |
| `Option<T>` + `bool` | `enum { Connected(T), Disconnected }` | 1 impossible state eliminated |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| Deserializable type with boxed string fields and builder conversions | Measure the serde/builder friction; retain the owned string type or convert once at a boundary if the guarantee does not justify the cost |
| `Box<[T]>` assumed to make every `T` immutable | State separately whether the container or its elements are frozen; inspect `&mut T` access |
| Newtype has no boundary behavior, validation, or domain methods | Prefer a type alias when arithmetic and interoperability dominate |
| Domain validation is repeated at call sites | Validate once and carry the validated type |
| Correlated flags and optional fields encode states | Replace them with an enum |

## Workflow
1. Find mutable representations whose post-construction capabilities are unused.
2. Check construction, public callers, serialization, builders, and measured hot paths.
3. Separate container immutability from element/content mutability.
4. Choose boxed storage, alias, newtype, `NonZero`, or enum based on the invariant and integration cost.
5. Report the representation change, tradeoff, and prevented bug class.

## Boundary

Use `rust-ecosystem` for dependency, feature, and build portability questions; use `zero-alloc` for measured hot-path allocation optimization. This skill decides whether a type-level invariant or immutability guarantee justifies its representation and integration cost.
