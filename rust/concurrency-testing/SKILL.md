---
name: concurrency-testing
description: "Design and review deterministic Rust concurrency tests that force and verify legal interleavings, safety invariants, histories, and bounded recovery. Use when testing TOCTOU windows, double or duplicate consumption, lost updates, order-dependent behavior, stale writes, cancellation and retry recovery, lost wakeups, ABA/version tokens, queue capacity or closure races, channels, locks, atomics, tasks, or logical race conditions. 中文触发：并发测试、竞态测试、时序依赖、TOCTOU、双重消费、重复消费、竞态条件、交错测试、并发不变量。"
---

# Rust Concurrency Testing

## Core Question

Determine what must remain true for every legal interleaving, then force the dangerous boundary and diagnose the violating actor, event, and state. Treat logical races such as stale decisions or duplicate effects separately from memory data races.

Use this skill for test design and review. Use `testing-strategy` for general test levels, fixtures, and determinism; use `async-concurrency` for production task ownership, cancellation, synchronization, and backpressure; use `resource-lifecycle` when cleanup or lease lifetime is the primary concern.

## Workflow

### 1. Define the contract and history

Write the invariant and allowed outcomes before choosing a barrier or scheduler:

- **Safety**: no lost update, stale write, invalid state, duplicate effect, or more than one claim for an item.
- **Consumption**: distinguish at-most-once, exactly-once, and idempotent retry semantics.
- **Ordering**: state required happens-before edges or an allowed outcome set; do not promote incidental executor order to a contract.
- **Progress**: define completion, bounded waiting, cancellation, and shutdown behavior where liveness matters.

Record actors, shared resources, observation and mutation points, intended linearization point, responses, and side effects. For operations with return values, check whether the concurrent history is equivalent to a legal linearizable or serializable history, as required by the contract, rather than checking only the final state.

When validity depends on state identity, include an ABA schedule such as A -> B -> A and assert that a generation, version, or lease token detects the intervening change.

Choose an oracle that matches the contract: a sequential reference model, a set of allowed outcomes, or an invariant/metamorphic relation. Do not require one exact final state when concurrent operations are allowed to commute.

### 2. Force the critical interleaving

Replace timing guesses with explicit handshakes. Gate the smallest boundary that exposes the bug:

| Failure class | Forced schedule | Required assertions |
|---|---|---|
| TOCTOU | A observes/checks; pause; B changes or claims; release A to commit | A rejects stale state or revalidates; no invalid mutation or effect occurs |
| Double consumption | N actors start with the same item or delivery; release them together | Claim/effect count matches the contract; losers return the specified conflict, skip, or retry result |
| Order dependence | Deliver valid permutations, duplicates, and permitted out-of-order events | Final state and emitted effects match or belong to the documented outcome set |
| Race/lost update | Synchronize reads, then release competing writes or commits | The invariant holds; no write, acknowledgement, or event is silently lost |
| Failure recovery | Pause after a side effect and before mark/ack; inject cancellation, panic, timeout, close, or restart | Retry behavior follows the contract: rollback, compensation, deduplication, or explicit at-least-once delivery |
| Lost wakeup | Signal before wait, wait before signal, and close/cancel while waiting | No signal is lost; every participant completes or returns the specified bounded outcome |
| Capacity/closure | Fill a bounded queue or exhaust permits; race send/receive with close or permit release | No item or permit is stranded; overload and closure return the specified outcome |

Use `Barrier`, channels, `Condvar`, or project-native async signals such as `Notify` and `oneshot`. Prefer a test seam, fake transport/store, injected pause, or controlled scheduler over changing production timing. Bound every wait so a failed participant cannot leave the test hanging.

For persistence or messaging behavior, add an integration test at the real transaction, unique-constraint, acknowledgement, or redelivery boundary. Do not let an in-memory mock claim guarantees that the database or broker does not provide.

### 3. Explore schedules deliberately

- Use one deterministic handshake test to prove each known vulnerable window.
- Enumerate a small state space of actor and event permutations when order is the risk.
- Use a model scheduler such as `loom` for small state spaces and modeled synchronization primitives; keep inputs bounded and assert the production contract.
- Use seeded property or fuzz tests for event sequences, retries, duplicate deliveries, and identifiers.
- Use repeated stress runs only as supplementary evidence. Passing many times does not prove an unforced schedule is safe.

For liveness, distinguish deadlock (no progress), livelock (activity without completion), and starvation (one actor is continually bypassed). Use controlled turns, resource limits, and bounded deadlines instead of scheduler luck. In async tests, await readiness signals rather than sleeping or polling; in thread tests, join every worker and preserve its result.

For atomics, model the weakest intended memory ordering and its publication/visibility edge. Assert what another actor may observe after synchronization, not only the eventual final value.

Do not use `sleep`, `yield_now`, test-thread count, or scheduler fairness as the only mechanism for reaching a schedule.

### 4. Assert, trace, and replay

Check all observable parts of the contract:

- final shared state and version or claim token;
- every actor's result, including stale-attempt and expected-loser errors;
- durable writes, acknowledgements, messages, callbacks, and external-effect counts;
- cancellation, timeout, closure, restart, and cleanup after a blocked or failed participant.

Give each actor, item, attempt, and idempotency key a unique identifier. Retain a compact trace such as `(actor, step, item, version)` with the seed, permutation, or schedule that failed. Make the smallest failing history replayable and shrink actor/event counts when property or model exploration finds a counterexample.

Do not reduce a concurrent test to `is_ok()`, a final count without per-actor outcomes, or a timeout with no diagnosis.

### 5. Isolate and clean up

Create fresh state per test and avoid process-global mutable fixtures. Ensure every gate can be released on error, cancellation, and panic; otherwise the first assertion failure can mask the real failure as a deadlock. Bound retries, close channels deliberately, and join or cancel spawned work before returning.

## Common Mistakes

| Weak test | Correction |
|---|---|
| `sleep` before a second operation | Pause at the exact check/commit or claim/ack boundary with a signal |
| Run the race many times and hope it appears | Force the interleaving, then add model or seeded exploration |
| Assert only the final value | Assert history or allowed outcome, per-actor results, invariant, version, and effect count |
| Assume one consumer means exactly-once | Specify at-most-once, exactly-once, or idempotent semantics and test retries |
| Require one worker or event order without a contract | Enumerate legal permutations and assert the allowed outcome set |
| Ignore a crash between effect and acknowledgement | Inject the failure and test rollback, compensation, deduplication, or redelivery |
| Leave a worker behind after timeout | Cancel or join it and verify resource, lease, and channel cleanup |
| Call every logical race a data race | Separate stale/duplicate/lost-update behavior from memory-model violations; use a model checker or sanitizer only where applicable |

## Reporting

For each missing or flaky test, report the invariant, legal history or forced schedule, synchronization seam, assertions, failure/cleanup path, replay artifact, and residual schedule coverage. Keep the test at the lowest level that observes the contract, adding integration coverage when persistence, transport, or task wiring changes the result.
