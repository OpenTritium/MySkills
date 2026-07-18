# Skill Trigger Matrix

Use this matrix to keep skill ownership explicit. Each query should have one primary skill; a secondary skill is acceptable only when the request crosses a documented boundary.

## Review And Refactoring

| Query | Primary skill | Secondary boundary |
|---|---|---|
| `use ConnectionState::*` causes unclear imports | `rust-import-hygiene` | `naming-smell` only for aliases |
| Simplify nested `if let` with `let-else` | `rust-guard-clauses` | `error-silence` for lost error context |
| Merge duplicate methods but preserve a test seam | `rust-api-consolidation` | `rust-structure-refactor` for broader decomposition |
| Decide whether an extracted Rust helper belongs on a type, in a newtype, or as a free function | `rust-method-placement` | `rust-structure-refactor` for broader decomposition; `encode-invariant` for newtype representation |
| AI-generated Rust refactor adds a small helper and its owner is unclear | `rust-method-placement` | `rust-structure-refactor` for the surrounding function or module split |
| Add method-like behavior to a Rust type that cannot receive an inherent method | `rust-method-placement` | `encode-invariant` when a local wrapper is a better owner |
| A local function has parameter explosion, hidden effects, or mixed abstraction levels | `func-smell` | `rust-structure-refactor` for broader decomposition; `rust-method-placement` for ownership |
| Boolean parameter is ambiguous at the call site | `func-smell` | `naming-smell` for the parameter name; keep a clear predicate expression when splitting would duplicate behavior |
| Replace `bool` plus `Option` with explicit states | `rust-state-machine` | `encode-invariant` for the type invariant |
| Split an 800-line function or module | `rust-structure-refactor` | `architecture-entropy-review` if ownership or routes multiply |
| Review a large refactor for duplicate owners | `architecture-entropy-review` | narrower smell skill for local evidence |
| Review a broad refactor in a Git or Jujutsu repository | `architecture-entropy-review` | `vcs-router` for backend selection and matching command group |
| Refactor or fix a bug while the behavior contract is uncertain | `testing-strategy` | `rust-structure-refactor` after happy/unhappy characterization or regression tests establish the contract |

## Local Code Quality

| Query | Primary skill | Secondary boundary |
|---|---|---|
| `try_` naming for fallible Rust methods | `naming-smell` | `error-silence` for the actual error contract |
| `tracing` event is too noisy | `high-snr-log` | `log-level` when severity is the question |
| Choose `INFO` versus `DEBUG` for retries | `log-level` | `high-snr-log` for payload noise |
| Remove comments that only restate code | `high-snr-comment` | none by default |
| Reduce nested loops or replace an unnecessary sort | `big-o-optimizer` | `zero-alloc` for allocation cost |
| Avoid allocations in a hot path | `zero-alloc` | `big-o-optimizer` for algorithmic cost |

## Language And Runtime

| Query | Primary skill | Secondary boundary |
|---|---|---|
| Production `Send`, cancellation, channel, lock, or deadlock design | `async-concurrency` | `concurrency-testing` only for deterministic behavioral coverage |
| Choose std versus async locks, atomics, channels, semaphores, or notifications | `async-concurrency` | `concurrency-testing` for behavioral coverage; `resource-lifecycle` for ownership and cleanup |
| Choose MPSC versus MPMC or sync versus async channels | `async-concurrency` | `concurrency-testing` for ordering/backpressure coverage; `rust-ecosystem` for crate integration |
| Choose a domain-specific concurrency crate instead of std/Tokio primitives | `async-concurrency` | `rust-ecosystem` for dependency compatibility and supply-chain review |
| Resource cleanup, `Drop`, locks, or transaction scope | `resource-lifecycle` |
| Encode a non-empty or immutable value in its type | `encode-invariant` |
| Choose boxed storage or alias versus newtype under serde, builder, arithmetic, or serialization integration constraints | `encode-invariant` | `rust-ecosystem` for dependency/build compatibility; `zero-alloc` for measured hot-path allocation cost |
| Handle `Result`, `unwrap`, or ignored errors | `error-silence` |
| Design Snafu errors, selectors, `Whatever`, or `context` | `rust-snafu` | `error-silence` for swallowing/logging; `rust-ecosystem` for dependency/features |
| Unsafe code, FFI, raw pointers, `unsafe impl Send/Sync`, or `SAFETY` | `unsafe-checker` |
| Test-only `set_var`/`remove_var` or audit a low-unsafe Rust codebase for hidden FFI and manual thread-safety assumptions | `unsafe-checker` | `testing-strategy` for general isolation; `concurrency-testing` when a deterministic interleaving is required |
| Cargo features, crate compatibility, or MSRV | `rust-ecosystem` |
| Unit, integration, property, characterization, regression, or general behavioral test design | `testing-strategy` | `async-concurrency` for production async design; `concurrency-testing` for forced interleavings |
| Force and verify TOCTOU, double-consumption, order-dependent, or race interleavings in tests | `concurrency-testing` | `testing-strategy` for general test design; `async-concurrency` for production concurrency design |
| Test cancellation/retry recovery, lost wakeups, ABA/version, or queue capacity/closure races | `concurrency-testing` | `async-concurrency` for the production lifecycle or synchronization design |

## Version Control

| Detect the repository backend before a status, diff, log, mutation, or workspace operation | `vcs-router` | `jujutsu` only when it returns `vcs=jj` |
| Run status, diff, log, mutation, or workspace operations after `vcs=jj` is selected | `jujutsu` | `vcs-router` for detection; `jujutsu-parallel` for parallel workspaces |
| Coordinate multiple agents in Jujutsu workspaces | `jujutsu-parallel` | `vcs-router` for detection and `jujutsu` for the base workflow |

## Maintenance Rules

- Add a query when a new skill is introduced or a trigger boundary changes.
- Remove or reassign a query when two skills claim the same primary responsibility.
- Prefer a narrow primary skill plus one explicit secondary over loading every related skill.
