---
name: zero-alloc
description: Use when reviewing memory allocation in Rust hot paths — heap allocations, iterator chains that collect, missing capacity, String temporaries, SmallVec/arrayvec, Cow, buffer reuse, boxing small structs, slice vs Vec signatures. Keywords: zero-alloc, allocation, heap, stack, GC, garbage collection, object pool, sync.Pool, iterator, generator, pre-allocate, capacity, StringBuilder, escape analysis, 零分配, 内存优化
---

# Zero-Alloc Performance Reviewer

## Overview
The fastest allocation is no allocation. In hot paths, eliminate heap allocs — keep data on the stack, borrow instead of clone, reuse buffers, use lazy iterators. Rust makes every `Vec`/`String`/`Box`/`to_string()` a visible heap trip. Goal: stack-keep hot-path data, borrow what you don't own.

## When to Use
- High-frequency functions allocating `Vec`/`String`/`Box` per call
- Loops building strings via `format!`/`to_string()`/`to_owned()`/`push_str`
- `.collect()` into a `Vec` iterated only once
- Small structs boxed or `&`-passed when they'd fit a register
- Latency-sensitive paths with allocator pressure in flamegraphs

## Optimization Rules

1. **Borrow, Don't Own** — `&str` over `String`, `&[T]` over `Vec<T>` in signatures. Return `&str`/`&[T]` views instead of cloning. A signature demanding ownership it never needs forces every caller to allocate.

2. **Reserve Capacity Up Front** — Known/estimable size: `Vec::with_capacity(n)` / `String::with_capacity(n)` before the loop. Without it, every `push`/`push_str` risks realloc + copy.

3. **Lazy Iterators Over `collect`** — Caller iterates once → return `impl Iterator<Item = T>` instead of `Vec`. O(1) vs O(N) memory. Collect only for random access, multiple passes, or ownership hand-off.

4. **Stack-Backed Replacements** — Fixed, small: `SmallVec<[T; N]>` / `ArrayVec<T, N>` (stack until spill). Fixed, known: `[T; N]` (zero allocs ever). `Cow<[T]>` / `Cow<str>` borrows when possible, allocs on mutation.

5. **Buffer Reuse** — Repeated work into the same scratch space: take `&mut Vec<T>` / `&mut String` (or wrap a buffer in your struct), `.clear()` between iterations. The allocator never sees it.

6. **Avoid Hidden Allocations**
   - `format!` allocs a `String` each call — in hot loops, `write!(buf, ...)` into a reusable `String`.
   - `.collect()` then `.iter()` again allocs just to re-iterate — chain instead.
   - `Box::new(SmallStruct{})` "to avoid a copy" forces a heap alloc — pass small structs by value.
   - `.cloned()` allocs per element on `Copy` types — use `.copied()` (no clone); borrow over `to_string`.
   - Closure capture can move (alloc) a `Vec`/`String` when `&` would do.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `fn find(ids: Vec<u64>, x: u64)` | `fn find(ids: &[u64], x: u64)` — borrow |
| `Vec::new()` then `push` in loop, known size | `Vec::with_capacity(n)` — no reallocs |
| `fn ids() -> Vec<u64>`, caller iterates once | Return `impl Iterator<Item=u64>` — zero alloc |
| `s += &format!("{}", x)` in a loop | `write!(s, "{}", x)` into reusable `String` |
| `Box::new(Point{x, y})` for 16 bytes | Pass `Point` by value — `Copy`, fits registers |
| `Vec<T>` almost always 0–8 elements | `SmallVec<[T; 8]>` — stack until spill |
| `.cloned()` on `&[u32]` | `.copied()` — no clone overhead |
| Fresh `String` per request in a handler | `&mut String` scratch on the struct, `.clear()` per request |

## Workflow
1. Identify the hot path (profiled or known: handler, parser loop, packet encode/decode)
2. List every alloc site: `Vec::new`, `to_string`/`format!`, `Box::new`, `.collect()`, `.clone()` on owned data
3. For each: borrow? reserve capacity? return iterator? stack (`SmallVec`/array)?
4. Per-call scratch → hoist to caller/struct, `.clear()` between calls
5. Output per change: alloc site → approach, removes entirely or reduces reallocs
6. Flag safety/clarity trades — buffer reuse must not leak state across calls; document it
