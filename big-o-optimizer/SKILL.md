---
name: big-o-optimizer
description: Use when reviewing Rust algorithm complexity — O(N²) nested loops, wrong data structures, redundant traversal, missing memoization, over-sorting, iterator misuse, sort vs select_nth_unstable, HashMap vs BTreeMap vs BinaryHeap. Keywords: Big-O, time complexity, space complexity, O(N), O(N²), O(log N), algorithm, data structure, HashMap, HashSet, BTreeMap, BinaryHeap, heap, Trie, memoization, DP, dynamic programming, sliding window, two pointers, sort_unstable, select_nth_unstable, partition_point, binary_search, iterator, fold, zip, windows, chunks, 算法优化, 时间复杂度
---

# Big-O Profiler & DSA Optimizer

## Overview
In large Big-O gaps, micro-optimization is futile. Optimize via data structure upgrades, loop elimination, caching — targeting O(N²) → O(N log N) → O(N) → O(1). Rust force-multipliers: fused lazy iterators, a precise sort toolkit (stable/unstable, partial select, cached keys), and zero-cost abstraction.

## When to Use
- Nested loops (intersection/association/join logic)
- `.contains()` / `.position()` on `Vec`/slices inside loops
- `sort()` on an entire collection when only top-K needed
- Multiple passes over the same data
- Recursion without memoization
- Hand-rolled loops duplicating an iterator combinator
- Choosing between sort variants, or `HashMap`/`BTreeMap`/`BinaryHeap`

## Optimization Rules

1. **Data Structure Upgrade**
   - Lookup/dedup: `Vec` (O(N)) → `HashSet`/`HashMap` (O(1))
   - Ordered lookup / range: linear scan → `BTreeMap`/`BTreeSet` (O(log N); `range()` is O(log N) + count)
   - Top-K / rolling min-max: `sort_by()` (O(N log N)) → `BinaryHeap` (O(log N) push/pop)
   - Prefix search: → `Trie` crate or `BTreeMap::range`

2. **Loop Unnesting (O(N²) → O(N))**
   - Pre-load inner-loop data into `HashMap` for O(1) lookup
   - Two pointers / sliding window on sorted data or contiguous sub-ranges
   - `.zip()`, `.enumerate()`, `.position()` replace manual inner loops

3. **Redundant Computation Elimination**
   - Hoist loop invariants
   - Single-pass multi-aggregation via `.fold()` (max+sum+count → one fold, not three passes)
   - Memoization/DP via `HashMap` keyed by argument

4. **Pruning & Early Termination**
   - `.find()` / `.any()` / `.position()` / `.all()` short-circuit — prefer over `.filter().next()` or a full loop
   - `binary_search` / `partition_point` on sorted data: O(log N)

## Data Structure Complexity Cheat Sheet

| Operation | `Vec`/`[T]` | `HashSet`/`HashMap` | `BTreeSet`/`BTreeMap` | `BinaryHeap` |
|---|---|---|---|---|
| Lookup / `contains` | O(N) | **O(1)** avg | O(log N) | O(N) |
| Insert | O(1) amortized | **O(1)** avg | O(log N) | O(log N) |
| Ordered iteration | O(N) | — | **O(N) sorted** | O(N log N) drained |
| Min / Max | O(N) | — | **O(log N)** (first/last) | O(1) peek (max *or* min, not both) |
| Range query | O(N) | — | **O(log N + k)** | — |
| Top-K | O(N log N) | — | O(K log N) | **O(K log K)** size-K heap |

> Heap gotcha: `BinaryHeap` is a max-heap with **no O(1) min** — use `Reverse` for a min-heap. Top-K smallest → size-K max-heap, evict largest; top-K largest → size-K min-heap, evict smallest.

## Sorting Method Selection

Six slice sorters + partial selection — NOT interchangeable:

| Need | Use | Cost | Notes |
|---|---|---|---|
| Default order | `sort_unstable` / `_by` | O(N log N), **in-place, no alloc** | Preferred unless equal-element order matters (pdqsort) |
| Stable (equal elements keep order) | `sort` / `sort_by` / `_by_key` | O(N log N), **allocs N temp** | Timsort; only when stability required |
| Sort by derived key | `sort_unstable_by_key` | O(N log N), no alloc | Key recomputed O(N log N) times — if expensive, `sort_by_cached_key` (O(N) evals) |
| Top-K / Kth element | `select_nth_unstable` | **O(N)** avg | Quickselect; then sort the K → O(N + K log K). Beats full sort for K ≪ N |
| Boundary in sorted data | `partition_point` / `binary_search_by` | O(log N) | No sort needed |
| Small arrays (≤ ~20) | `sort_unstable_by` | ~O(N²) worst | pdqsort insertion-sort fast path for tiny slices |

> Two biggest wins: (a) `sort_unstable_*` over `sort_*` when stability isn't needed — 1.5–2× faster, no alloc. (b) `select_nth_unstable` over full `sort` for top-K — O(N log N) → O(N).

## Advanced Iterator Patterns

`.map().filter().fold()` is **one fused O(N) pass**, not three. Prefer combinators over hand-rolled loops — they fuse, short-circuit, and document intent.

| Pattern | Combinator | Replaces |
|---|---|---|
| Multi-aggregate single pass | `.fold()` tuple accumulator | 3 separate `for` loops |
| Element + index | `.enumerate()` | manual `i += 1` counter |
| Two collections parallel | `.zip()` O(min(a,b)) | nested loop O(a×b) |
| Sliding window | `.windows(n)` | manual `i`/`i+n` index math |
| Non-overlapping chunks | `.chunks(n)` / `chunks_exact(n)` | manual stride |
| Every k-th element | `.step_by(k)` | `if i % k == 0` |
| Group equal runs | `.chunk_by()` (1.77+) / itertools `group_by` | manual group state |
| Dedup consecutive | `.dedup()` / `dedup_by()` | compare-with-previous |
| Global dedup | `HashSet::from_iter` | nested `contains` |
| Early-exit | `.try_fold()`, `.find()`, `.position()` | `loop { ...; break }` |
| Take/skip prefix | `.take(n)` / `.skip(n)` + `_while` variants | counter + break/continue |
| Flatten nested | `.flatten()` / `flat_map()` | nested `for` |
| Scan with state | `.scan(state, f)` | external `let mut state` |
| Filter + transform | `.filter_map()` | `if pred { push }` into temp Vec |

Anti-patterns iterators kill:
- **Collect-then-reloop**: `.collect()` then iterate again — chain combinators, collect once (or never).
- **Manual short-circuit**: `for` + `break` where `.any()`/`.find()` expresses it and documents the exit.
- **Index math for windows**: `for i in 0..len-n` is off-by-one prone; `.windows(n)` is correct.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `vec.contains(&x)` in a loop | `HashSet` before the loop: O(N²) → O(N) |
| `vec.iter().position(...)` in a loop | Build `HashMap<T, usize>` once, O(1) lookup |
| `sort_by()` then `[..k]` for top-K | `select_nth_unstable(k)` → O(N) not O(N log N) |
| `sort()` where stability irrelevant | `sort_unstable()` — faster, no alloc |
| `sort_by_key(\|x\| expensive(x))` | `sort_by_cached_key` — O(N) key evals |
| Linear `contains` on sorted data | `binary_search` / `partition_point` — O(log N) |
| Two passes for max + sum + count | Single `.fold()` |
| `retain`/`filter` chains reallocating each | One iterator pass |
| `.collect::<Vec<_>>()` then `.iter()` again | Chain; collect once or never |
| `for x in v { if pred { out.push(f(x)) } }` | `.filter_map(\|x\| ...).collect()` |
| Top-K via full-N `BinaryHeap` | Size-K heap → O(N log K) |
| `v.sort();` before one `binary_search` | Few lookups → linear scan faster; many → sort once, reuse |
| Rolling max via re-scan per window | Monotonic `VecDeque` → O(N) not O(N·k) |

## Workflow
1. State current Big-O (time + space), highlight offending lines
2. Project at N=10,000 and N=1,000,000
3. Data structure: does the cheat sheet offer O(1)/O(log N) where you do O(N)?
4. Sort: `sort` where `sort_unstable` suffices? Full sort where `select_nth_unstable` gives top-K in O(N)?
5. Loops: can combinators fuse passes or express a short-circuit?
6. Output before/after complexity table (time + space)
