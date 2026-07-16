---
name: resource-lifecycle
description: 'Review Rust resource ownership and cleanup across RAII, Drop, pools, transactions, leases, background workers, and shutdown. Use when code opens connections/files, manages guards or locks, starts background tasks, commits transactions, or needs graceful cleanup. 中文触发：资源生命周期、RAII、Drop、连接池、事务、优雅关闭'
---

# Resource Lifecycle Reviewer

## Core Rules

1. **Name the owner** — Every resource has one owner, a transfer/borrow rule, and a cleanup path on success, error, cancellation, and early return.
2. **Use the right cleanup boundary** — RAII guards handle synchronous release; fallible or async shutdown needs an explicit operation that callers can await and observe.
3. **Keep `Drop` safe** — Avoid blocking, allocation-heavy, fallible, or panic-prone work in `Drop`; use explicit close/shutdown for errors and async work.
4. **Pool and lease discipline** — Bound capacity and wait time; return or invalidate leases on every path; define behavior during pool shutdown and task cancellation.
5. **Transaction semantics** — Make commit, rollback, retry, and ownership visible; ensure cancellation cannot silently commit or leave an unusable transaction.
6. **Background work** — Store task handles, stop producers before consumers, await joins, and release channels, sockets, and permits during shutdown.

## Review Process

1. Draw ownership and lifetime from acquisition to release.
2. Trace normal, error, panic, cancellation, and shutdown paths.
3. Inspect guards, `Drop`, `mem::forget`, `Arc` cycles, pools, and transaction boundaries.
4. Report leaks, double release, hidden blocking, lost shutdown errors, and unclear ownership separately.

## Common Mistakes

| Finding | Correction |
|---|---|
| Async cleanup hidden in `Drop` | Expose `close`/`shutdown` and await its result |
| Lease not returned on an early error | Use a guard or one ownership path for every exit |
| `mem::forget` used to silence cleanup | Prove the intentional transfer or remove it |
| Background task has no shutdown signal | Own its handle and cancellation path |
| Pool shutdown races with new acquisitions | Close admission before draining active users |
| `Drop` logs or panics on routine failure | Make cleanup infallible or report it through explicit shutdown |
