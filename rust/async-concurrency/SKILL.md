---
name: async-concurrency
description: 'Review Rust async and concurrency design for task ownership, cancellation, synchronization, backpressure, and blocking. Use when code uses async/await, Tokio, threads, Send/Sync, channels, locks, spawn, timeout, select, shutdown, or deadlock. 中文触发：异步、并发、取消、死锁、任务、通道'
---

# Async Concurrency Reviewer

## Core Rules

1. **Task ownership** — Every spawned task has an owner, a join or cancellation path, and an error policy. Avoid untracked background work.
2. **Cancellation** — Make cancellation and timeout behavior explicit; do not leave partial state, leases, or transactions behind. Check whether awaited operations are cancellation-safe.
3. **Synchronization** — Choose `std` or async-aware primitives by blocking behavior. Do not hold locks across `.await` unless required; document lock ordering and protect against deadlock.
4. **Executor health** — Keep blocking CPU or I/O off async workers; use an appropriate blocking boundary and bound concurrency.
5. **Channels and backpressure** — Prefer bounded queues when producers can outrun consumers; handle send failure, closure, fairness, and overload explicitly.
6. **Send/Sync contracts** — Treat `Send`/`Sync` bounds and shared mutable state as design constraints, not compiler obstacles.

## Review Process

1. Draw task, resource, and shutdown ownership.
2. Trace success, error, timeout, cancellation, and executor-shutdown paths.
3. Check every spawn, lock, channel, retry loop, and blocking call.
4. Report races, deadlocks, starvation, leaks, unbounded work, and lost task errors separately.

## Common Mistakes

| Finding | Correction |
|---|---|
| `spawn` handle discarded | Store, await, abort, or explicitly detach with a lifecycle owner |
| `std::sync::Mutex` held across `.await` | Shorten the critical section or use an async primitive when waiting is required |
| Unbounded channel for request work | Add capacity and define overload behavior |
| `sleep` used as synchronization | Await a condition, signal, or bounded timeout |
| Blocking file/CPU work on the executor | Move it behind a blocking boundary and bound parallelism |
| Timeout returns while a child task continues | Cancel or join the child and release its resources |
