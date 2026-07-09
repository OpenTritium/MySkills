---
name: big-o-optimizer
description: Use when reviewing algorithm complexity — O(N²) nested loops, wrong data structures, redundant traversal, missing memoization, over-sorting. Keywords: Big-O, time complexity, space complexity, O(N), O(N²), O(log N), algorithm, data structure, HashMap, HashSet, heap, Trie, memoization, DP, dynamic programming, sliding window, two pointers, 算法优化, 时间复杂度
---

# Big-O Profiler & DSA Optimizer

## Overview
In large Big-O gaps, micro-optimization is futile. Optimize via data structure upgrades, loop elimination, and caching — targeting O(N²) → O(N log N) → O(N) → O(1). Sacrifice memory for speed when warranted.

## When to Use
- Nested loops over collections (especially intersection/association logic)
- Frequent `.contains()` / `.indexOf()` calls on Lists inside loops
- Sorting entire collection when only top-K elements needed
- Multiple passes over the same data for different aggregations
- Heavy recursion without memoization

## Optimization Rules

1. **Data Structure Upgrade**
   - Frequent lookup/dedup: `List` (O(N)) → `HashSet`/`HashMap` (O(1))
   - Frequent top-K / dynamic min-max: `Sort` (O(N log N)) → `PriorityQueue`/Heap (O(log N) per op)
   - Prefix/range search: linear scan → `Trie` or `TreeMap`/B-Tree (O(log N))

2. **Loop Unnesting (O(N²) → O(N))**
   - Space-time tradeoff: pre-load inner-loop data into a Hash table for O(1) lookup
   - Two pointers / sliding window: if data is sorted or seeking contiguous sub-ranges

3. **Redundant Computation Elimination**
   - Hoist loop invariants outside loops
   - Single-pass multi-aggregation instead of multiple traversals
   - Memoization/DP for overlapping recursive subproblems

4. **Pruning & Early Termination**
   - `break`/`return` immediately when target found
   - Bound checks to skip impossible search branches

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `list.contains(x)` inside a loop | Convert inner list to `Set` outside the loop |
| `list.insert(0, item)` repeatedly | Use `LinkedList`/`ArrayDeque`, or reverse-append then reverse once |
| Full `sort()` for top-10 of a million items | Use a size-10 `PriorityQueue` (Heap) |
| Multiple `for` loops over same collection for max, min, avg | Single pass collecting all metrics |

## Workflow
1. State current Big-O (time and space), highlight the offending lines
2. Project behavior at N=10,000 and N=1,000,000 (CPU spike, timeout risk)
3. Apply optimization rules, produce refactored code
4. Output before/after complexity comparison table
