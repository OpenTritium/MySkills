# Skill Trigger Matrix

Use this matrix to keep skill ownership explicit. Each query should have one primary skill; a secondary skill is acceptable only when the request crosses a documented boundary.

## Review And Refactoring

| Query | Primary skill | Secondary boundary |
|---|---|---|
| `use ConnectionState::*` causes unclear imports | `rust-import-hygiene` | `rust-naming-smell` only for aliases |
| Simplify nested `if let` with `let-else` | `rust-guard-clauses` | `rust-error-silence` for lost error context |
| Merge duplicate methods but preserve a test seam | `rust-api-consolidation` | `rust-structure-refactor` for broader decomposition |
| Decide whether an extracted Rust helper belongs on a type, in a newtype, or as a free function | `rust-method-placement` | `rust-structure-refactor` for broader decomposition; `rust-encode-invariant` for newtype representation |
| AI-generated Rust refactor adds a small helper and its owner is unclear | `rust-method-placement` | `rust-structure-refactor` for the surrounding function or module split |
| Add method-like behavior to a Rust type that cannot receive an inherent method | `rust-method-placement` | `rust-encode-invariant` when a local wrapper is a better owner |
| A local function has parameter explosion, hidden effects, or mixed abstraction levels | `rust-func-smell` | `rust-structure-refactor` for broader decomposition; `rust-method-placement` for ownership |
| Boolean parameter is ambiguous at the call site | `rust-func-smell` | `rust-naming-smell` for the parameter name; keep a clear predicate expression when splitting would duplicate behavior |
| Replace `bool` plus `Option` with explicit states | `rust-state-machine` | `rust-encode-invariant` for the type invariant |
| Split an 800-line function or module | `rust-structure-refactor` | `rust-architecture-entropy-review` if ownership or routes multiply |
| `pub(crate)` is scattered through implementation modules and the intended boundary is unclear | `rust-structure-refactor` | `rust-api-consolidation` when existing internal APIs can be merged or removed |
| Organize multiple test-only methods on a Rust struct with `#[cfg(test)]` | `rust-structure-refactor` | `rust-testing-strategy` when choosing the unit versus integration-test boundary |
| Review a large refactor for duplicate owners | `rust-architecture-entropy-review` | narrower smell skill for local evidence |
| Review a broad refactor in a Git or Jujutsu repository | `rust-architecture-entropy-review` | `vcs-router` for backend selection and matching command group |
| Refactor or fix a bug while the behavior contract is uncertain | `rust-testing-strategy` | `rust-structure-refactor` after happy/unhappy characterization or regression tests establish the contract |

## Local Code Quality

| Query | Primary skill | Secondary boundary |
|---|---|---|
| `try_` naming for fallible Rust methods | `rust-naming-smell` | `rust-error-silence` for the actual error contract |
| `tracing` event is too noisy, repetitive, oversized, or missing useful fields | `rust-logging-review` | `rust-tracing-context` for async correlation |
| Choose `INFO` versus `DEBUG` for retries | `rust-logging-review` | `rust-tracing-context` for propagation across async boundaries |
| Review logs from `tokio::spawn` that lose task identity | `rust-tracing-context` | `rust-async-concurrency` for ownership/cancellation; `rust-logging-review` for log payload context |
| Review `instrument`, `in_current_span`, `or_current`, or async span propagation | `rust-tracing-context` | `rust-logging-review` for filter severity; `rust-async-concurrency` for task lifecycle |
| Decide whether `task_local!` is sufficient for correlating Rust async logs | `rust-tracing-context` | `rust-async-concurrency` for task-local ownership and cancellation |
| Remove comments that only restate code | `rust-high-snr-comment` | none by default |
| Reduce nested loops or replace an unnecessary sort | `rust-big-o-optimizer` | `rust-zero-alloc` for allocation cost |
| Avoid allocations in a hot path | `rust-zero-alloc` | `rust-big-o-optimizer` for algorithmic cost |

## External API Design

| Query | Primary skill | Secondary boundary |
|---|---|---|
| Design or review a public HTTP/JSON resource API | `stripe-api-design` | `rust-method-placement` or `rust-structure-refactor` for Rust ownership and module structure |
| Define idempotent writes, cursor pagination, structured errors, or API versioning | `stripe-api-design` | `rust-error-silence` or `rust-snafu` for Rust error implementation; `rust-testing-strategy` for behavioral coverage |
| Design or review webhook/event delivery semantics | `stripe-api-design` | `rust-async-concurrency` for task ownership and cancellation; `rust-concurrency-testing` for deterministic duplicate/order/retry tests |
| Review competing API sources of truth across modules or routes | `rust-architecture-entropy-review` | `stripe-api-design` for the external wire contract |

## Go

| Query | Primary skill | Secondary boundary |
|---|---|---|
| Review Go control flow and readability | golang-code-style | golang-naming for identifiers; golang-lint for enforcement |
| Choose Go package, type, function, or test names | golang-naming | golang-code-style for surrounding clarity; golang-documentation for exported docs |
| Write or review godoc, README, examples, or release documentation | golang-documentation | golang-naming for exported names; golang-testing for executable examples |
| Create, wrap, inspect, or log a Go error | golang-error-handling | golang-safety for nil/error-value hazards |
| Review Go code for nil, aliasing, numeric, resource, or zero-value bugs | golang-safety | golang-security only for attacker-controlled impact; golang-concurrency for races |
| Design Go structs, methods, receivers, embedding, or interfaces | golang-structs-interfaces | golang-design-patterns for broader architecture; golang-naming for names |
| Configure or interpret golangci-lint, go vet, or staticcheck | golang-lint | golang-code-style for the rule target; golang-security for security-only audits |
| Upgrade old Go idioms, standard-library calls, or toolchains | golang-modernize | golang-dependency-management for module changes; golang-lint for static rules |
| Write or review Go unit, integration, fuzz, or race tests | golang-testing | golang-concurrency for production behavior; keep Testify-specific choices inside this skill when the project already uses it |
| Audit Go code for vulnerabilities, secrets, injection, or threat paths | golang-security | golang-safety for internal correctness; golang-lint for scanner configuration |
| Design goroutines, channels, locks, atomics, worker pools, or fan-out | golang-concurrency | golang-context for cancellation; golang-safety for race-prone state |
| Propagate Go context, deadlines, cancellation, or request values | golang-context | golang-concurrency when goroutines or task ownership are involved |
| Choose or optimize Go slices, maps, containers, builders, or pointer storage | golang-data-structures | golang-performance for measured hot paths; golang-safety for aliasing |
| Write or review Go database queries, scans, pools, or transactions | golang-database | golang-security for query threats; golang-error-handling for failures; golang-context for cancellation |
| Choose manual DI or introduce do/v2 in a Go application | golang-dependency-injection | golang-structs-interfaces for interface boundaries; golang-context for lifecycle |
| Choose Go constructors, options, lifecycle, resilience, or architecture patterns | golang-design-patterns | golang-structs-interfaces for type design; golang-project-layout for repository structure |
| Start or reorganize a Go module, package tree, workspace, or CLI layout | golang-project-layout | golang-design-patterns for architecture; golang-dependency-injection for wiring |
| Refactor existing Go code across functions, types, packages, or callers | golang-refactoring | golang-gopls for semantic mechanics; target rules belong to naming/style/layout skills |
| Write or compare Go benchmarks, profiles, or benchstat reports | golang-benchmark | golang-performance for the fix; golang-troubleshooting for root cause |
| Set up Go GitHub Actions, releases, dependency updates, or CI quality gates | golang-continuous-integration | golang-lint, golang-security, and golang-dependency-management for individual gates |
| Add Go logs, metrics, traces, profiling, dashboards, or alerts | golang-observability | golang-performance for a measured bottleneck; golang-security for sensitive telemetry |
| Apply a Go optimization after identifying a bottleneck | golang-performance | golang-benchmark for measurement; golang-concurrency for synchronization cost |
| Debug a Go crash, race, deadlock, leak, or unexpected result | golang-troubleshooting | golang-safety for common pitfalls; golang-concurrency for task behavior; golang-benchmark for performance diagnosis |
| Build or review a Go CLI's command tree, flags, signals, or exit codes | golang-cli | golang-project-layout for repository structure; golang-context for cancellation |
| Add, upgrade, audit, or explain Go module dependencies | golang-dependency-management | golang-security for vulnerabilities |
| Navigate local Go symbols, references, diagnostics, or semantic refactors | golang-gopls | golang-refactoring for the overall change process |
| Design or implement a Go GraphQL schema, resolver, subscription, or data loader | golang-graphql | golang-testing for behavior; golang-error-handling for failures |
| Design or implement Go gRPC services, protobufs, interceptors, streams, or TLS | golang-grpc | golang-testing for bufconn/stream tests; golang-security for transport/auth |
| Design, generate, or review OpenAPI 3.x APIs with ogen | golang-openapi | golang-documentation for API docs; golang-security for auth and threat review |

## Language And Runtime

| Query | Primary skill | Secondary boundary |
|---|---|---|
| Production `Send`, cancellation, channel, lock, or deadlock design | `rust-async-concurrency` | `rust-concurrency-testing` only for deterministic behavioral coverage |
| Choose std versus async locks, atomics, channels, semaphores, or notifications | `rust-async-concurrency` | `rust-concurrency-testing` for behavioral coverage; `rust-resource-lifecycle` for ownership and cleanup |
| Choose MPSC versus MPMC or sync versus async channels | `rust-async-concurrency` | `rust-concurrency-testing` for ordering/backpressure coverage; `rust-ecosystem` for crate integration |
| Choose a domain-specific concurrency crate instead of std/Tokio primitives | `rust-async-concurrency` | `rust-ecosystem` for dependency compatibility and supply-chain review |
| Resource cleanup, `Drop`, locks, or transaction scope | `rust-resource-lifecycle` |
| Encode a non-empty or immutable value in its type | `rust-encode-invariant` |
| Choose boxed storage or alias versus newtype under serde, builder, arithmetic, or serialization integration constraints | `rust-encode-invariant` | `rust-ecosystem` for dependency/build compatibility; `rust-zero-alloc` for measured hot-path allocation cost |
| Handle `Result`, `unwrap`, or ignored errors | `rust-error-silence` |
| Design Snafu errors, selectors, `Whatever`, or `context` | `rust-snafu` | `rust-error-silence` for swallowing/logging; `rust-ecosystem` for dependency/features |
| Unsafe code, FFI, raw pointers, `unsafe impl Send/Sync`, or `SAFETY` | `rust-unsafe-checker` |
| Test-only `set_var`/`remove_var` or audit a low-unsafe Rust codebase for hidden FFI and manual thread-safety assumptions | `rust-unsafe-checker` | `rust-testing-strategy` for general isolation; `rust-concurrency-testing` when a deterministic interleaving is required |
| Cargo features, crate compatibility, or MSRV | `rust-ecosystem` |
| Unit, integration, property, characterization, regression, or general behavioral test design | `rust-testing-strategy` | `rust-async-concurrency` for production async design; `rust-concurrency-testing` for forced interleavings |
| Force and verify TOCTOU, double-consumption, order-dependent, or race interleavings in tests | `rust-concurrency-testing` | `rust-testing-strategy` for general test design; `rust-async-concurrency` for production concurrency design |
| Test cancellation/retry recovery, lost wakeups, ABA/version, or queue capacity/closure races | `rust-concurrency-testing` | `rust-async-concurrency` for the production lifecycle or synchronization design |

## Version Control

| Detect the repository backend before a status, diff, log, mutation, or workspace operation | `vcs-router` | `jujutsu` only when it returns `vcs=jj` |
| Run status, diff, log, mutation, or workspace operations after `vcs=jj` is selected | `jujutsu` | `vcs-router` for detection; `jujutsu-parallel` for parallel workspaces |
| Inspect a Jujutsu working-copy revision before committing; split mixed logical changes by exact filesets and validate the resulting graph | `jujutsu` | `vcs-router` for detection; `jujutsu-parallel` for multiple workspaces |
| Coordinate multiple agents in Jujutsu workspaces | `jujutsu-parallel` | `vcs-router` for detection and `jujutsu` for the base workflow |

## .NET

| Query | Primary skill | Secondary boundary |
|---|---|---|
| Add or review .NET 10 ASP.NET Core controller/minimal API endpoints, DTOs, OpenAPI metadata, or ProblemDetails | `dotnet-webapi` | `stripe-api-design` for the language-neutral resource and wire contract; a persistence skill for storage details |
| Design resource semantics, idempotency, pagination, versioning, or webhook behavior for a .NET 10 ASP.NET Core API | `stripe-api-design` | `dotnet-webapi` for C# endpoint and middleware implementation |
| Run, filter, or diagnose the command used to execute .NET 10 TUnit tests | `run-tests` | `writing-tunit-tests` for test source |
| Write or modernize .NET 10 TUnit assertions, data-driven cases, lifecycle, or async tests | `writing-tunit-tests` | `run-tests` for execution; a test-audit skill for broad quality findings |
| Resolve .NET 10 trimming or Native AOT analyzer warnings | `dotnet-aot-compat` | none |
| Audit or fix .NET 10 .csproj, .props, .targets, or Directory.Build MSBuild anti-patterns | `msbuild-antipatterns` | `dotnet-aot-compat` only for AOT-related project properties |
| Analyze .NET 10 C# allocation, async, LINQ, regex, serialization, or I/O performance patterns | `analyzing-dotnet-performance` | use an algorithm skill for complexity |

## Maintenance Rules

- Add a query when a new skill is introduced or a trigger boundary changes.
- Remove or reassign a query when two skills claim the same primary responsibility.
- Prefer a narrow primary skill plus one explicit secondary over loading every related skill.
