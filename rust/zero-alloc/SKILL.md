---
name: zero-alloc
description: 'Review avoidable heap allocation in Rust hot paths: owned parameters, temporary strings, repeated collection, missing capacity, buffer reuse, SmallVec/ArrayVec, Cow, boxing, and slice-versus-Vec signatures. Keywords: zero-alloc, allocation, heap, stack, iterator, pre-allocate, capacity, format!, collect, SmallVec, ArrayVec, Cow, buffer reuse, 零分配, 内存优化'
---

# Zero-Alloc Performance Reviewer

## Overview
In hot paths, remove avoidable heap allocations: borrow where ownership is unnecessary, reuse buffers, and avoid temporary collections. Confirm allocation claims with profiling; not every ownership move or clone allocates.

## Optimization Rules

1. **Borrow, Don't Own** — `&str` over `String`, `&[T]` over `Vec<T>` in signatures. Return `&str`/`&[T]` views instead of cloning. A signature demanding ownership it never needs forces every caller to allocate.

2. **Reserve Capacity Up Front** — Known/estimable size: `Vec::with_capacity(n)` / `String::with_capacity(n)` before the loop. Without it, every `push`/`push_str` risks realloc + copy.

3. **Lazy Iterators Over `collect`** — When the caller only needs one pass, return an iterator or view instead of an eagerly owned `Vec`. This can avoid a caller-owned collection allocation; collect for random access, multiple passes, or ownership hand-off.

4. **Stack-Backed Replacements** — Fixed, small: `SmallVec<[T; N]>` / `ArrayVec<T, N>` (stack until spill). Fixed, known: `[T; N]` (zero allocs ever). `Cow<[T]>` / `Cow<str>` borrows when possible, allocs on mutation.

5. **Buffer Reuse** — Repeated work into the same scratch space: take `&mut Vec<T>` / `&mut String` (or wrap a buffer in your struct), `.clear()` between iterations. The allocator never sees it.

6. **Avoid Hidden Allocations**
   - `format!` allocs a `String` each call — in hot loops, `write!(buf, ...)` into a reusable `String`.
   - `.collect()` then `.iter()` again allocs just to re-iterate — chain instead.
   - `Box::new(SmallStruct{})` adds a heap allocation — prefer a value when ownership, layout, and measured ABI costs allow.
   - `.copied()` makes `Copy` intent explicit; `.cloned()` invokes `Clone` and may allocate when the element type's `Clone` does.
   - `move` capture changes ownership; inspect async/task boxing and captured values separately when diagnosing allocations.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `fn find(ids: Vec<u64>, x: u64)` | `fn find(ids: &[u64], x: u64)` — borrow |
| `Vec::new()` then `push` in loop, known size | `Vec::with_capacity(n)` — no reallocs |
| `fn ids() -> Vec<u64>`, caller iterates once | Return an iterator/view when it avoids an owned collection — measure |
| `s += &format!("{}", x)` in a loop | `write!(s, "{}", x)` into reusable `String` |
| `Box::new(Point{x, y})` for 16 bytes | Pass `Point` by value — `Copy`, fits registers |
| `Vec<T>` almost always 0–8 elements | `SmallVec<[T; 8]>` — stack until spill |
| `.cloned()` used only for `Copy` intent | `.copied()` — clearer intent; allocation depends on `Clone` implementation |
| Fresh `String` per request in a handler | `&mut String` scratch on the struct, `.clear()` per request |

## Workflow
1. Identify the hot path (profiled or known: handler, parser loop, packet encode/decode)
2. List every alloc site: `Vec::new`, `to_string`/`format!`, `Box::new`, `.collect()`, `.clone()` on owned data
3. For each: borrow? reserve capacity? return iterator? stack (`SmallVec`/array)?
4. Per-call scratch → hoist to caller/struct, `.clear()` between calls
5. Output per change: alloc site → approach, removes allocation or reduces reallocations
6. Flag safety/clarity trades — buffer reuse must not leak state across calls; document it
