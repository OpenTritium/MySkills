---
name: zero-alloc
description: Use when reviewing memory allocation in Rust hot paths — heap allocations, iterator chains that collect, missing capacity, String temporaries, SmallVec/arrayvec, Cow, buffer reuse, boxing small structs, slice vs Vec signatures. Keywords: zero-alloc, allocation, heap, stack, GC, garbage collection, object pool, sync.Pool, iterator, generator, pre-allocate, capacity, StringBuilder, escape analysis, 零分配, 内存优化
---

# Zero-Alloc Performance Reviewer

## Overview
The fastest allocation is no allocation. In hot paths and tight loops, eliminate unnecessary heap allocations — keep data on the stack, borrow instead of clone, reuse buffers, and use lazy iterators. Rust makes allocations visible: every `Vec`, `String`, `Box`, and `to_string()` is a heap trip. The goal is to keep hot-path data on the stack and borrow what you don't own.

## When to Use
- High-frequency functions allocating temporary `Vec`/`String`/`Box` on every call
- Loops with `format!`/`to_string()`/`to_owned()` or `push_str` building strings
- `.collect()` into a `Vec` that the caller only iterates once
- Small structs passed by `Box`/`&` when they'd fit in a register
- Latency-sensitive paths where allocator pressure shows up in flamegraphs

## Optimization Rules

1. **Borrow, Don't Own** — Take `&str` over `String`, `&[T]` over `Vec<T>` in function signatures. Return `&str`/`&[T]` views into owned data instead of cloning. A signature that asks for ownership it never needs forces every caller to allocate.

2. **Reserve Capacity Up Front** — When the size is known or estimable, `Vec::with_capacity(n)` / `String::with_capacity(n)` before the loop. Without it, every `push`/`push_str` risks a realloc + copy. `Itertools::collect_vec` doesn't know the size hint's upper bound is real; `with_capacity` does.

3. **Lazy Iterators Over `collect`** — If the caller only iterates once, return an iterator (`impl Iterator<Item = T>`) instead of a `Vec`. O(1) memory instead of O(N). Only `.collect()` when you need random access, multiple passes, or ownership hand-off.

4. **Stack-Backed Replacements** — Fixed-size, small: `SmallVec<[T; N]>` / `ArrayVec<T, N>` allocate on the stack until they spill. Fixed-size, known: `[T; N]` is a stack array, zero allocations ever. `Cow<[T]>` / `Cow<str>` returns a borrowed view when possible, allocates only on mutation.

5. **Buffer Reuse** — For repeated work into the same scratch space, take a `&mut Vec<T>` / `&mut String` parameter (or wrap a reusable buffer in your struct) and `.clear()` between iterations instead of allocating fresh each call. The allocator never sees it.

6. **Avoid Hidden Allocations**
   - `format!` allocates a `String` every call — in a hot loop, write into a reusable `String` with `write!(buf, ...)`.
   - `.collect()` then `.iter()` again allocates a `Vec` just to re-iterate — chain iterators instead.
   - `Box::new(SmallStruct{})` "to avoid a copy" forces a heap alloc; pass small structs by value.
   - Iterator chains with `.cloned()` / `.copied()` / `.map(|x| x.to_string())` allocate per element — prefer `.copied()` over `.cloned()` for `Copy` types (no per-element clone), and borrow over `to_string`.
   - Closure capture can force a `Vec`/`String` to move (allocate) when a `&` would do.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `fn find(ids: Vec<u64>, x: u64)` | `fn find(ids: &[u64], x: u64)` — caller keeps owning, you borrow |
| `let mut v = Vec::new(); for ... { v.push(x) }` with known size | `let mut v = Vec::with_capacity(n)` — no reallocs |
| `fn ids() -> Vec<u64>` built once, caller does `for x in ids() { ... }` | Return `impl Iterator<Item=u64>` — zero allocation if caller only iterates |
| `s += &format!("{}", x)` inside a loop | `write!(s, "{}", x)` into a reusable `String`; reuse across iterations |
| `Box::new(Point{x, y})` for a 16-byte struct | Pass `Point` by value — it's `Copy`, fits in registers |
| `Vec<T>` that's almost always 0–8 elements | `SmallVec<[T; 8]>` — stack until spill, no allocator pressure in the common case |
| `.cloned()` over a `&[u32]` | `.copied()` — `Copy` types, no clone overhead |
| Fresh `String` built per request in a handler | Take `&mut String` scratch buffer on the struct, `.clear()` per request |

## Workflow
1. Identify the hot path (profiled or known by design — request handler, parser inner loop, packet encode/decode)
2. List every allocation site: `Vec::new`, `String::from`/`to_string`/`format!`, `Box::new`, `.collect()`, `.clone()` on owned data
3. For each, ask: can it borrow instead of own? Can it reserve capacity? Can it return an iterator? Can it live on the stack (`SmallVec`/array)?
4. For per-call scratch, hoist the buffer to the caller or a long-lived struct and `.clear()` between calls
5. Output the diff with, for each change: old allocation site → new approach, and whether it removes the allocation entirely or only reduces reallocs
6. Flag any change that trades safety/clarity for allocation — e.g. raw buffer reuse must not leak state across calls; document the invariant
