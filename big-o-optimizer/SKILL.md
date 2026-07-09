---
name: big-o-optimizer
description: Use when reviewing Rust algorithm complexity — O(N²) nested loops, wrong data structures, redundant traversal, missing memoization, over-sorting, iterator misuse, sort vs select_nth_unstable, HashMap vs BTreeMap vs BinaryHeap. Keywords: Big-O, time complexity, space complexity, O(N), O(N²), O(log N), algorithm, data structure, HashMap, HashSet, BTreeMap, BinaryHeap, heap, Trie, memoization, DP, dynamic programming, sliding window, two pointers, sort_unstable, select_nth_unstable, partition_point, binary_search, iterator, fold, zip, windows, chunks, 算法优化, 时间复杂度
---

# Big-O Profiler & DSA Optimizer

## Overview
In large Big-O gaps, micro-optimization is futile. Optimize via data structure upgrades, loop elimination, and caching — targeting O(N²) → O(N log N) → O(N) → O(1). Sacrifice memory for speed when warranted. Rust gives you three force-multipliers most languages don't: (1) a rich iterator combinator library that fuses passes and stays lazy, (2) a precise sorting toolkit (stable vs unstable, partial select, cached keys), and (3) zero-cost abstraction so the idiomatic form is usually also the fast one.

## When to Use
- Nested loops over collections (especially intersection/association/join logic)
- Frequent `.contains()` / `.position()` / `.rposition()` calls on `Vec`/slices inside loops
- `sort()` / `sort_by()` on an entire collection when only the top-K elements are needed
- Multiple passes over the same data for different aggregations
- Heavy recursion without memoization
- Hand-rolled loops that duplicate what an iterator combinator does in one fused pass
- Choosing between sort variants, or between `HashMap`/`BTreeMap`/`BinaryHeap`

## Optimization Rules

1. **Data Structure Upgrade**
   - Frequent lookup/dedup: `Vec` (O(N) `contains`) → `HashSet`/`HashMap` (O(1))
   - Frequent ordered lookup / range queries: linear scan → `BTreeMap`/`BTreeSet` (O(log N), and `range()` is O(log N) + count)
   - Top-K / rolling min-max: `sort_by()` (O(N log N)) → `BinaryHeap` (O(log N) per push/pop)
   - Prefix search: linear scan → a `Trie` crate or `BTreeMap::range`

2. **Loop Unnesting (O(N²) → O(N))**
   - Space-time tradeoff: pre-load inner-loop data into a `HashMap` for O(1) lookup (the classic "join via map" pattern)
   - Two pointers / sliding window: if data is sorted or you seek contiguous sub-ranges
   - Iterator combinators: `.zip()`, `.enumerate()`, `.position()` often replace a manual inner loop or a manual index counter

3. **Redundant Computation Elimination**
   - Hoist loop invariants outside loops
   - Single-pass multi-aggregation via `.fold()` instead of multiple traversals (one pass for max, another for sum, another for count → one fold)
   - Memoization/DP for overlapping recursive subproblems — `HashMap` keyed by the argument

4. **Pruning & Early Termination**
   - `.find()` / `.any()` / `.position()` / `.all()` short-circuit immediately when the target is found — prefer them over `.filter().next()` or a manual loop that runs to completion
   - `binary_search` / `partition_point` on a sorted slice: O(log N) instead of O(N) lookup
   - Bound checks to skip impossible search branches

## Data Structure Complexity Cheat Sheet

| Operation | `Vec`/`[T]` | `HashSet`/`HashMap` | `BTreeSet`/`BTreeMap` | `BinaryHeap` |
|---|---|---|---|---|
| Lookup / `contains` | O(N) | **O(1)** avg | O(log N) | O(N) (heap is unordered!) |
| Insert | O(1) amortized push | **O(1)** avg | O(log N) | O(log N) push |
| Ordered iteration | O(N) (contiguous) | — (unordered) | **O(N) sorted** | O(N log N) to drain sorted |
| Min / Max | O(N) scan | — | **O(log N)** (first/last) | O(1) peek (min *or* max, not both) |
| Range query | O(N) | — | **O(log N + k)** via `range()` | — |
| Top-K | O(N log N) sort | — | O(K log N) | **O(K log K)** with size-K heap |

> Heap gotcha: `BinaryHeap` is a max-heap and has **no O(1) min**. For a min-heap wrap values in `Reverse`. For "top-K smallest", keep a max-heap of size K and evict the largest; for "top-K largest", keep a min-heap and evict the smallest.

## Sorting Method Selection

Rust has six slice sorters plus partial-selection — they are NOT interchangeable. Pick by need:

| You need... | Use | Cost | Notes |
|---|---|---|---|
| Default sorted order | `sort_unstable` / `sort_unstable_by` | O(N log N), **in-place, no alloc** | Preferred unless equal elements' order matters. Pattern-defeating quicksort. |
| Equal elements keep original order | `sort` / `sort_by` / `sort_by_key` | O(N log N), **allocs N temp** | Stable (Timsort-style). Slower; only when stability is required. |
| Sort by a derived key | `sort_unstable_by_key` | O(N log N), no alloc | Key recomputed O(N log N) times — if key is expensive, use `sort_by_cached_key` to memoize it to O(N). |
| Only the top-K / "Kth element" | `select_nth_unstable` | **O(N)** avg | Quickselect partition; then `sort_unstable` the K you keep → O(N + K log K). Beats full sort for K ≪ N. |
| Find boundary in already-sorted data | `partition_point` / `binary_search_by` | O(log N) | No sort at all — the data is already ordered. |
| Sort small arrays (≤ ~20) | `sort_unstable_by` | ~O(N²) worst but fast | Unstable pdqsort has an insertion-sort fast path for tiny slices. |

> The two most common wins: (a) `sort_unstable_*` over `sort_*` when you don't need stability — free, often 1.5–2× faster, no allocation. (b) `select_nth_unstable` over full `sort` + slice for top-K — turns O(N log N) into O(N).

## Advanced Iterator Patterns

Rust iterators are lazy: `.map().filter().fold()` is **one fused O(N) pass**, not three. Reach for combinators before a hand-rolled loop — they fuse, they short-circuit, and they document intent.

| Pattern | Combinator | Replaces (manual) |
|---|---|---|
| Single pass, multiple aggregates | `.fold()` with a tuple accumulator | 3 separate `for` loops (3× O(N) → 1× O(N)) |
| Element + index | `.enumerate()` | `let mut i = 0; for x in ... { ...; i += 1 }` |
| Parallel iteration of two collections | `.zip()` — O(min(a,b)) | nested loop O(a×b) |
| Sliding window of size n | `.windows(n)` | manual `i`/`i+n` index math |
| Non-overlapping chunks | `.chunks(n)` / `.chunks_exact(n)` | manual stride loop |
| Every k-th element | `.step_by(k)` | `if i % k == 0` |
| Group consecutive equal runs | `.chunk_by()` (1.77+) / itertools `.group_by` | manual "is this a new group?" state |
| Remove consecutive duplicates | `.dedup()` / `.dedup_by()` | manual compare-with-previous |
| Global dedup (unordered) | `HashSet::from_iter` then iterate | nested `contains` |
| Early-exit fold/find | `.try_fold()`, `.try_for_each()`, `.find()` | `loop { ...; break }` |
| Take prefix | `.take(n)`, `.take_while()` | counter + `break` |
| Skip prefix | `.skip(n)`, `.skip_while()` | counter + `continue` |
| Flatten nested | `.flatten()`, `.flat_map()` | nested `for` |
| Scan with state | `.scan(state, f)` | external `let mut state` |
| Conditional running logic | `.filter()`, `.filter_map()` | `if ... { push }` into temp `Vec` |

Anti-patterns that iterators specifically kill:
- **Collect-then-reloop**: `.collect::<Vec<_>>()` then iterate again. If you only needed one pass, chain the combinators and collect once at the end (or never).
- **Manual short-circuit**: a `for` loop with `break` that `.any()`/`.find()`/`.position()` expresses directly — and the combinator name *documents* the early exit.
- **Index math for windows**: `for i in 0..v.len()-n { &v[i..i+n] }` is an off-by-one waiting to happen; `.windows(n)` is correct and free.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `vec.contains(&x)` inside a loop | Collect into `HashSet` before the loop: O(N²) → O(N) |
| `vec.iter().position(...)` inside a loop | Build `HashMap<T, usize>` index once, then O(1) lookup |
| `vec.sort_by(...)` then take `vec[..k]` for top-K of a million items | `select_nth_unstable(k)` then sort the K → O(N) instead of O(N log N) |
| `sort()` where order of equal elements is irrelevant | `sort_unstable()` — faster, no allocation |
| `sort_by_key(\|x\| expensive(x))` recompute | `sort_by_cached_key` — memoize the key to O(N) evaluations |
| Linear `vec.contains` on sorted data | `slice::binary_search` / `partition_point` — O(log N) |
| `let mut mins = ...; let mut maxs = ...; for x in v { ... }` two passes | Single `.fold()` collecting all metrics |
| Repeated `retain`/`filter` chains that each reallocate | Fuse into one iterator pass, or one `fold`/`filter` |
| `.collect::<Vec<_>>()` then `.iter()` again | Chain combinators; collect once (or never) |
| `for x in v { if pred { out.push(f(x)) } }` | `v.iter().filter_map(\|x\| ...).collect()` |
| Top-K via `BinaryHeap` of all N elements | Size-K heap, push+pop the rest → O(N log K) not O(N log N) |
| `v.sort();` before a one-off `binary_search` | If only a few lookups, linear scan is faster; if many, sort once then reuse |
| Rolling max via re-scan each window | Monotonic `VecDeque` deque, or `.windows()` + tracking — O(N) not O(N·k) |

## Workflow
1. State current Big-O (time and space), highlight the offending lines
2. Project behavior at N=10,000 and N=1,000,000 (CPU spike, timeout risk)
3. Check the data structure: does the cheat sheet offer an O(1)/O(log N) operation where you currently do O(N)?
4. Check the sort: is it `sort` (stable, allocs) where `sort_unstable` would do? Full sort where `select_nth_unstable` gives top-K in O(N)?
5. Check the loops: can iterator combinators fuse multiple passes into one, or express a short-circuit?
6. Apply the optimization, produce refactored code
7. Output before/after complexity comparison table (time + space)
