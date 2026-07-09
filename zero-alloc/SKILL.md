---
name: zero-alloc
description: Use when reviewing memory allocation in hot paths — heap allocations, GC pressure, object pooling, iterator patterns, pre-allocation, escape analysis, string concatenation in loops. Keywords: zero-alloc, allocation, heap, stack, GC, garbage collection, object pool, sync.Pool, iterator, generator, pre-allocate, capacity, StringBuilder, escape analysis, 零分配, 内存优化
---

# Zero-Alloc Performance Reviewer

## Overview
The fastest allocation is no allocation. In hot paths and tight loops, eliminate unnecessary heap allocations — keep data on the stack, reuse buffers, and use lazy evaluation.

## When to Use
- High-frequency functions allocating temporary objects/buffers on every call
- Loops with `+=` string concatenation or append-without-capacity
- Functions returning fully-materialized large collections for iteration
- Small structs passed by pointer causing heap escape
- GC pause investigation in latency-sensitive systems

## Optimization Rules

1. **Lazy Evaluation & Iterators** — Replace `List<T>` return types with `Iterator<T>`/generator/`yield` when caller only needs to iterate. O(1) memory instead of O(N).

2. **Object Pooling & Buffer Reuse** — Use `sync.Pool` (Go), `ThreadLocal`/ObjectPool (Java), or workspace structs to reuse frequently-allocated objects across calls.

3. **Pre-allocation & Sizing** — When the final or approximate size is known, initialize with capacity: `make([]int, 0, capacity)`, `new ArrayList<>(capacity)`, `StringBuilder(capacity)`. Eliminates hidden reallocation and copy costs.

4. **Escape Analysis & Stack Allocation** — Avoid returning pointers to small local structs. Pass small structs by value. Avoid unnecessary interface boxing that forces heap allocation.

5. **Zero-Copy Strings & Bytes** — Use `StringBuilder`/`Buffer` for loop concatenation. Use slice views (`string_view`, `&str`) instead of copying when mutation is not needed.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `findAllUserIds()` returns `List<Long>` with millions of IDs | Return `Iterator<Long>` or accept a callback |
| `s += item` inside a loop | `StringBuilder` with pre-allocated capacity outside the loop |
| `list.append(x)` in loop without capacity | Initialize with `make/capacity` before the loop |
| Pointer to small struct (`Point{x, y}`) to "avoid copy" | Pass by value — avoids heap escape and GC pressure |
