---
name: rust-tracing-context
description: 'Review Rust asynchronous tracing context propagation and task correlation across `tokio::spawn`, `tracing::Instrument`, spans, task-local context, and OpenTelemetry log bridges. Use when logs from spawned tasks lose their task identity, when reviewing `instrument`, `in_current_span`, `or_current`, `#[tracing::instrument]`, or `task_local!`, or when checking whether CDC/worker logs are associated with the correct task. 中文触发：tracing 上下文、日志关联、任务关联、子任务日志、task local、task_local、异步 span、tokio::spawn、链路上下文'
---

# Rust Tracing Context Reviewer

## Core Question

Determine whether every event emitted during asynchronous work is correlated with the span of its owning task, including work that crosses `.await`, `tokio::spawn`, child spans, filtering, and exporter layers. Preserve business identity explicitly where it is needed; tracing context is an observability mechanism, not a replacement for domain data.

## Scope And Boundaries

- Use this skill for the shape and propagation of tracing context in Rust async code.
- Use `rust-async-concurrency` for task ownership, cancellation, synchronization, executor health, and backpressure. Bring it in when a context review finds a lifecycle defect.
- Use `rust-logging-review` for noisy, redundant, oversized, or context-poor log records.
- Use `rust-logging-review` for `ERROR`/`WARN`/`INFO`/`DEBUG`/`TRACE` severity decisions.
- Use `rust-architecture-entropy-review` when the issue is a cross-module source of truth or execution-route split rather than span propagation.

Do not turn this into a general logging or concurrency review. State the narrower boundary in findings when another skill owns the primary defect.

## Review Workflow

1. Draw the async ownership graph: the task entry point, each `spawn`, the owner of each `JoinHandle`, and the operations that emit events.
2. Identify the root span for each long-lived task. Record its stable fields, creation boundary, and whether the future is actually instrumented when it is spawned.
3. Trace every `.await` and task boundary. Check `in_current_span()`, `instrument(...)`, and `#[tracing::instrument]` at the point where the future is polled, not merely where it is constructed.
4. Inspect named child spans, filter behavior, task-local scopes, subscriber layers, and the exporter that ultimately writes logs or traces.
5. Separate findings into lost context, misleading context, duplicated context, and unrelated lifecycle or severity defects.
6. Validate with a focused test or captured subscriber output, then run the narrowest relevant Rust checks.

## Propagation Rules

### Establish One Root Span

Create a root span at the boundary that owns the task. Put stable correlation fields there, such as a task identifier, connector identifier, or worker name, while excluding credentials and whole configuration objects.

```rust
let task_span = tracing::info_span!("task.run", task_id = %task_id);
tokio::spawn(run_task(task).instrument(task_span));
```

If `#[tracing::instrument]` creates the root span at the task function boundary, do not add a second span with the same semantic ownership merely for visual nesting. Verify which boundary is intended to own the task.

### Instrument The Future

Use the `tracing::Instrument` extension on the future that crosses the async boundary:

```rust
use tracing::Instrument;

tokio::spawn(worker_future.instrument(task_span));
```

Use `.in_current_span()` when the future should inherit the span that is current at the spawn site:

```rust
tokio::spawn(worker_future.in_current_span());
```

Use `#[tracing::instrument]` for functions whose arguments and lifecycle naturally define a span. Add `skip(...)` for large or sensitive arguments and add explicit fields for stable identifiers.

### Preserve The Parent When A Child Is Filtered

For a named child span, prefer `.or_current()` before `.instrument(...)`:

```rust
let poll_span = tracing::debug_span!("source.poll", table = %table);
tokio::spawn(
    async move { poll_source().await }
        .instrument(poll_span.or_current()),
);
```

`or_current()` makes the current parent span the fallback when the named span is disabled by filtering. Without that fallback, an event in the child future can lose the task's root context even though an explicit `task_id` field elsewhere makes the output look correlated. Treat this as especially important when the root span is enabled at `INFO` and operational child spans are commonly filtered at `DEBUG`.

### Never Enter A Span Across `.await`

Do not hold a `Span::enter()` guard while awaiting. A guard can remain associated with the wrong execution context when futures are suspended and resumed on different tasks or workers. Instrument the future instead:

```rust
async fn run() {
    let span = tracing::info_span!("task.run");
    do_work().instrument(span).await;
}
```

If synchronous code needs a short nested event, scope the guard so it is dropped before any `.await`.

## Task-Local Context

Treat `tokio::task_local!` as typed task state, not as automatic tracing propagation.

- A task-local value is available only inside its explicit scope.
- A new `tokio::spawn` task does not inherit the parent task-local value automatically; initialize or pass it deliberately.
- A task-local value does not create span fields and does not cause a log exporter to attach `task_id`.
- Use task locals for typed request/task context when their scope and lifetime are clear. Use tracing spans for log and trace correlation.

When the same identity is needed by both mechanisms, initialize them independently at the task boundary. Do not remove an explicit domain argument merely because a task-local or span contains a similar value.

## Observability Versus Domain Identity

Keep these concerns separate:

| Need | Source of truth |
|---|---|
| Correlate logs and spans across async work | tracing span context |
| Label a metric series | explicit metric labels, usually `task_id` or a bounded task key |
| Address state in a database, etcd, or a queue | explicit domain identity |
| Carry retry, ownership, or business state | typed domain data |

Do not fix missing log association by interpolating `task_id` into every message. First establish the task root span and instrument all async routes. Keep explicit fields when the consumer is a metric, persistence key, API response, or exporter that does not inherit span fields.

## Exporter And Subscriber Checks

Inspect the actual subscriber and exporter before changing fields:

1. Determine whether the log layer records active span fields, event fields, both, or neither.
2. Determine whether a bridge copies root-span fields to leaf records, and whether it copies fields only when the child span is enabled.
3. Confirm filter directives for the root and child targets. A frequent low-value event belongs in `DEBUG` or `TRACE` based on operator value; severity is separate from propagation.
4. Check labels and field cardinality before adding task identifiers to metrics or logs at high volume.
5. Verify that logs emitted before task creation, from independent control-plane tasks, or during process startup are not falsely assigned to a CDC task.

Do not assume `tracing-opentelemetry` alone exports OpenTelemetry Logs. Confirm the configured Logs bridge/layer and backend ingestion path. A log backend showing no `level` or task field may indicate a layer or schema mapping problem rather than missing context in the Rust task.

## Spawn Review Checklist

For every `tokio::spawn`, `spawn_blocking`, or equivalent task boundary, answer all of these:

- Which owner starts and stops the task?
- Which root or current span should the future inherit?
- Is the future wrapped with `instrument(...)` or `in_current_span()` at spawn time?
- If it has a named, filterable child span, does it use `or_current()`?
- Is the `JoinHandle` awaited, aborted, supervised, or deliberately detached by a documented owner?
- What happens on error, cancellation, timeout, channel closure, and executor shutdown?
- Can the task produce unbounded events or queue work faster than its consumer?

The last three questions belong primarily to `rust-async-concurrency`; include them to prevent a context fix from hiding a task-lifecycle defect.

## Reusable Pattern

If a codebase repeatedly spawns futures in the current context, centralize the pattern so new call sites cannot omit instrumentation:

```rust
use std::future::Future;
use tokio::task::JoinHandle;
use tracing::Instrument;

fn spawn_in_current_span<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
{
    tokio::spawn(future.in_current_span())
}
```

Use a named helper only when the codebase has one consistent policy for ownership and error handling. Do not hide cancellation, joining, or supervision behind a helper that merely makes the call site look instrumented.

## Validation

Add or run focused checks that prove behavior, not just compilation:

1. Start a parent task span with a stable identifier and spawn a child future.
2. Capture a subscriber record from the child event and assert the identifier is present through the configured layer.
3. Filter the named child span while keeping the parent enabled; assert the child event still has the parent correlation when `.or_current()` is required.
4. Exercise nested `.await` points and a spawned grandchild rather than testing only a synchronous block.
5. If logs are exported to a backend, run an integration check against the actual bridge/schema and distinguish missing `level`, missing task fields, and missing records.

Report the first broken boundary with its file and line, the observable impact, and the smallest fix. A passing `cargo check` does not prove tracing context propagation.
