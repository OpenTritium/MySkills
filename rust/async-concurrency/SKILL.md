---
name: async-concurrency
description: 'Review Rust production async and concurrency design for task ownership, cancellation, synchronization primitive choice, backpressure, blocking, and domain-specific concurrency crate selection. Use when choosing async tasks, locks, channels, atomics, timeouts, shutdown, or deadlock behavior; use concurrency-testing for deterministic interleaving tests and testing-strategy for general test design. 中文触发：异步设计、并发设计、取消、死锁、任务、通道、锁原语、标准库 Mutex、Tokio Mutex、MPSC、MPMC、并发库选择'
---

# Async Concurrency Reviewer

## Core Rules

1. **Task ownership** — Every spawned task has an owner, a join or cancellation path, and an error policy. Avoid untracked background work.
2. **Cancellation** — Make cancellation and timeout behavior explicit; do not leave partial state, leases, or transactions behind. Check whether awaited operations are cancellation-safe.
3. **Synchronization** — Choose `std` or async-aware primitives by blocking behavior. Document lock ordering and protect against deadlock.
4. **Guard lifetime** — Do not hold lock or state-borrowing guards across `.await`; scope or drop them before awaiting, including guards that are not `Send`.
5. **Poisoning** — Treat lock poisoning as an explicit failure path; propagate it or deliberately recover rather than silently ignoring it.
6. **Executor health** — Keep blocking CPU or I/O off async workers; use an appropriate blocking boundary and bound concurrency.
7. **Channels and backpressure** — Prefer bounded queues when producers can outrun consumers; handle send failure, closure, fairness, and overload explicitly.
8. **Send/Sync contracts** — Treat `Send`/`Sync` bounds and shared mutable state as design constraints, not compiler obstacles.

## Primitive Selection

Choose a primitive based on where waiting occurs and who owns the state, not merely because the caller is async:

| Situation | Prefer | Boundary |
|---|---|---|
| Short, bounded critical section with no `.await` or blocking work | `std::sync::Mutex` or `std::sync::RwLock` | Bound contention; lock acquisition can still block an executor worker, especially on a current-thread runtime |
| Exclusive access must span an `.await` | Redesign to release before waiting; use `tokio::sync::Mutex` only when spanning the await is inherent | An async lock avoids blocking the worker while waiting for the lock, but can serialize tasks and complicate cancellation and fairness |
| One task can own all mutable state | Channel or actor ownership | Bound the queue and define shutdown, overload, and request cancellation |
| One independent scalar or flag | Atomic with explicit ordering | Do not use atomics to protect a multi-field invariant without a complete protocol |
| Limit concurrent work rather than protect state | Semaphore | Test permit release on success, error, cancellation, and shutdown |
| Notify that state may have changed | `Notify` plus condition re-check | `Notify` is not an event queue; use a channel when every event must be delivered |
| Blocking synchronous I/O or CPU work | `spawn_blocking` or a dedicated thread | Do not hold an async guard while entering or awaiting the blocking boundary |

Before choosing `tokio::sync::Mutex`, ask whether state can be copied out, an operation can be split around `.await`, or ownership can move to one task. Use an async lock only when asynchronous waiting while holding exclusive access is part of the real resource protocol.

## Selecting A Specialized Library

Do not maintain a crate ranking. Select a candidate from its public API, then verify performance with its benchmark source and the project's workload:

1. **State the contract** — Name producer/consumer topology, sync versus async waiting, boundedness, ordering, fairness, closure, cancellation, allocation, target, and MSRV requirements.
2. **Read the crate root** — Inspect `src/lib.rs` or its rendered crate documentation first. Confirm public constructors, sender/receiver ownership, `Send`/`Sync`/`Clone` bounds, blocking versus async methods, close/disconnect behavior, feature gates, and the actual queue or lock semantics. Follow re-exports only as far as needed to verify the contract.
3. **Check integration cost** — Read `Cargo.toml`, feature definitions, `cfg` target gates, MSRV declarations, transitive dependencies, and runtime coupling. Use `rust-ecosystem` for compatibility, maintenance, license, and supply-chain review.
4. **Find the benchmark evidence** — Locate `benches/`, `criterion`/other harness setup, benchmark configuration, and the commit or version being measured. Record producer/consumer counts, payload size, queue bound, runtime, CPU target, warmup, and whether allocation, cloning, drops, shutdown, and error paths are included.
5. **Reproduce before choosing** — Run the candidate's benchmark under the documented conditions, then add a realistic project benchmark for throughput, latency, tail latency, allocation, cancellation, closure, and overload. Treat README rankings and synthetic microbenchmarks as hypotheses, not proof.
6. **Prefer the smallest semantic fit** — Keep the standard or runtime primitive when it satisfies the contract. Add a specialized crate only when the measured workload or target platform exposes a real gap.

## Review Process

1. Draw task, resource, and shutdown ownership.
2. Trace success, error, timeout, cancellation, and executor-shutdown paths.
3. Check every spawn, lock, channel, retry loop, and blocking call.
4. For each primitive or external crate, record the wait location, critical-section bound, possible contention, runtime context, cancellation behavior, sync/async callers, API evidence, and benchmark evidence.
5. Report races, deadlocks, starvation, leaks, unbounded work, and lost task errors separately.

## Common Mistakes

| Finding | Correction |
|---|---|
| `spawn` handle discarded | Store, await, abort, or explicitly detach with a lifecycle owner |
| Tokio mutex chosen by default for async state | Prefer a short non-await standard-lock section or ownership transfer; use an async lock only when waiting across `.await` is inherent |
| Domain crate added for popularity or an unverified benchmark | Prove the semantic or platform gap, inspect the benchmark source, then reproduce it with the project workload |
| README benchmark ranking treated as a guarantee | Check the harness, inputs, runtime, target, and omitted work before drawing a conclusion |
| Lock or state-borrowing guard held across `.await` | Scope or drop it before awaiting; if waiting while holding it is inherent, review async-lock ownership and cancellation |
| Standard lock used under unknown or long contention | Bound the critical section or move blocking work behind `spawn_blocking`/a dedicated thread |
| Poisoned lock ignored | Propagate or deliberately handle the poisoning outcome |
| Unbounded channel for request work | Add capacity and define overload behavior |
| `sleep` used as synchronization | Await a condition, signal, or bounded timeout |
| Blocking file/CPU work on the executor | Move it behind a blocking boundary and bound parallelism |
| Timeout returns while a child task continues | Cancel or join the child and release its resources |
