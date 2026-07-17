---
name: rust-state-machine
description: "Review Rust state modeling and transitions with enums or typestate, preventing invalid states and incomplete transition handling. Use when working with state enums, status flags, `Option` plus booleans, state-dependent methods, protocol phases, or transition tests. 中文触发：状态机、状态转换、enum 状态、typestate、非法状态、状态流转"
---

# Rust State Machine

Model each valid state explicitly and make legal transitions easy to find. Preserve runtime flexibility when states come from external input; use compile-time types only when the protocol is stable enough to justify them.

## Rules

1. **Encode mutually exclusive states** — Replace related flags and options such as `(connected: bool, socket: Option<Socket>)` with an enum whose variants carry only the data valid in that state.

2. **Centralize transitions** — Put state changes beside the state and its invariant. Expose named operations such as `connect`, `authenticate`, or `close`; do not let arbitrary setters mutate state fields around the transition rules.

3. **Choose the type level deliberately** — Use an enum for runtime state, external input, persistence, or dynamic workflows. Use typestate when invalid call order should fail at compile time and the state set is small and stable. Do not introduce generic typestate layers that duplicate behavior or complicate public APIs.

4. **Make matching exhaustive** — List every domain variant when each state has meaningful behavior. Do not use `_` to hide a state that should trigger review when a new variant is added.

5. **Define invalid transitions** — Return a typed error, `Option`, or an explicit no-op only when that behavior is part of the contract. Do not silently ignore an invalid transition or collapse distinct failure causes.

6. **Preserve ownership and cleanup** — Check resource creation, release, rollback, lock scope, cancellation, and `.await` boundaries at every transition. Move state and its protected data together; do not leave a resource in a field that no longer represents its state.

7. **Test the transition matrix** — Cover each valid transition, representative invalid transitions, state data preservation, error behavior, terminal states, and cleanup. Test behavior through the transition API rather than adding test-only state mutators.

## Workflow

1. List states, state-specific data, events, legal transitions, terminal states, and failure outcomes.
2. Check whether flags or `Option` fields permit impossible combinations.
3. Choose an enum or typestate representation and assign one owner for transitions.
4. Implement named transitions and exhaustive state handling without broad wildcard matches.
5. Validate the transition matrix, public API, ownership, async behavior, and resource lifecycle.

```rust
enum Connection {
    Disconnected,
    Connecting { attempt: u32 },
    Ready { socket: Socket },
}

impl Connection {
    fn start(self) -> Result<Self, Error> {
        match self {
            Self::Disconnected => Ok(Self::Connecting { attempt: 0 }),
            Self::Connecting { .. } | Self::Ready { .. } => {
                Err(Error::InvalidTransition)
            }
        }
    }
}
```

Use `encode-invariant` for the type-level representation of an invariant, `rust-guard-clauses` for local early exits, and `resource-lifecycle` or `async-concurrency` for deeper cleanup and task ownership review.

Report the invalid state or transition, smallest safe model change, affected callers, and transition tests added or checked.
