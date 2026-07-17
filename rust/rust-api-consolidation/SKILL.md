---
name: rust-api-consolidation
description: "Decide which Rust methods, wrappers, and test-only entry points to merge or remove while preserving real seams and avoiding god methods. Use when reducing API surface, unifying similar operations, deleting test-only APIs, or reviewing a refactor that combines methods. 中文触发：API 合并、接口收敛、测试专用 API、seam、上帝方法、重复包装器"
---

# Rust API Consolidation

Reduce duplicate semantic entry points without hiding distinct contracts. Judge an API by its owner, invariant, error policy, lifecycle, side effects, and caller intent.

## Decide First

| Evidence | Action |
|---|---|
| Wrapper only forwards to another method and adds no domain meaning | Remove or inline it |
| Same owner, invariant, error policy, lifecycle, and side effects | Merge behind one canonical API |
| Different authorization, transaction, async, ownership, performance, or caller intent | Keep separate |
| No production caller; exists only to let tests reach internals | Delete it; test behavior or colocate a unit test |
| Boundary around production I/O, time, randomness, or external services | Keep a narrow seam |
| Compatibility or deprecation wrapper | Keep temporarily with an explicit migration boundary |

## Boundaries

- **One owner** — Put the canonical operation beside the invariant it protects. Do not merge domain, adapter, and orchestration APIs because their names look similar.
- **One abstraction level** — Do not turn a merged method into a god method spanning parsing, policy, persistence, retries, formatting, and unrelated state transitions.
- **No flag-driven merge** — If consolidation needs `bool` flags or a catch-all options struct to select different contracts, keep separate names or introduce a typed command.
- **Real seams only** — A seam must represent a production substitution or boundary that tests can exploit. Do not add a trait merely to mock pure logic; extract and test that logic directly.
- **Narrow visibility** — Keep the canonical API private until callers prove otherwise. Use `pub` or `pub(crate)` for a real contract, never to rescue test access or a privacy error.
- **Preserve contracts** — Compare return types, error context, side effects, ordering, cancellation, locks, transactions, visibility, and performance before changing callers.

## Procedure

1. Map callers and record each candidate's contract, owner, and test-only consumers.
2. Classify candidates using the table; choose one canonical owner and signature.
3. Merge one semantic group, remove obsolete wrappers and test-only APIs, and retain only real seams.
4. Run formatting, compile checks, focused tests, and public API or documentation checks as applicable.

```rust
fn fetch(&self, id: Id) -> Result<Item, Error> {
    self.load(id)
}
```

Remove this wrapper only when `fetch` and `load` have identical contracts. Keep separate methods when `fetch` adds cache or retry policy; the different names then carry useful semantics. Keep an injected `Clock`, repository, or transport when it models a production boundary and provides deterministic substitution.

Report the contract comparison, canonical owner, removed or retained seam, smallest safe change, and validation. Use `rust-structure-refactor` when consolidation requires broader function, struct, or module decomposition.
