---
name: rust-logging-review
description: 'Review Rust logging and tracing for signal quality, structured context, severity, filtering, telemetry cost, and sensitive-data exposure. Use when logs are noisy, repetitive, oversized, misleading, incorrectly leveled, or when choosing between events, spans, metrics, and OpenTelemetry output. Keywords: log, logger, logging, tracing, print, info, warn, error, debug, trace, log level, ERROR, WARN, INFO, DEBUG, TRACE, structured logging, OpenTelemetry, OTLP, 日志, 日志级别, 日志分级'
---

# Rust Logging Review

## Core question

Does each emitted record answer an operational question with the right context, severity, volume, and privacy boundary?

## Signal And Context

1. **Remove anti-echo logs** — Delete messages that only restate a method name or control flow. Use `#[tracing::instrument]` or a span for work boundaries instead of manual entering/leaving breadcrumbs.
2. **Add operational context** — Keep stable identifiers, state transitions, counts, latency, attempts, and reasons that let an operator distinguish entities and outcomes. If no operational question can be named, delete the record.
3. **Prefer structured fields** — Use indexable fields over interpolated `format!` text. Keep messages concise and avoid whole structs, request bodies, JSON, secrets, PII, and unbounded-cardinality values at every level; `TRACE` is not a privacy boundary.
4. **Avoid duplicate reporting** — Do not log and propagate the same error at multiple layers. Log once at the handling boundary; intermediate layers add context and return unless they contribute distinct action.
5. **Separate spans from events** — Use a span for a period of work and correlation; use an event for a point-in-time outcome. Do not hold a `Span::enter()` guard across `.await`.

| Signal to keep | Example |
|---|---|
| Failure with context | `error!(order_id, attempt, "gateway timeout")` |
| State transition | `info!(node_id, peers, "elected leader")` |
| Recoverable anomaly | `warn!(user_id, "rate limit hit")` |
| Decision and reason | `info!(request_id, "routed to fallback")` |
| Performance breach | `warn!(endpoint, p99_latency_ms, "breached SLO")` |

## Severity And Volume

- Treat a level as semantic priority and verbosity, not message length or field count. The order is `ERROR < WARN < INFO < DEBUG < TRACE`.
- Use `ERROR` for broken contracts, lost work, corruption, or invariant violations; `WARN` for hazards or degradation that still permits progress; `INFO` for useful normal state changes; `DEBUG` for lower-priority diagnostic decisions or recovery; and `TRACE` for very verbose execution-flow evidence.
- Choose severity by significance first. Control volume independently with filtering, sampling, aggregation, rate limiting, metrics, or removal. Do not downgrade `ERROR` merely to make telemetry quieter.
- Expected retry attempts usually belong at `DEBUG`; actual degradation belongs at `WARN`; terminal impact is classified by its consequence. Suppress repetition separately from severity.
- Prefer a span, metric, attribute, exception, or status when it represents the signal better than a repeated event. Avoid one `INFO` event per request or item in a hot path.

## Tracing And OpenTelemetry

- Inspect the complete route: `tracing` macros → subscriber/layers → `tracing-opentelemetry` → OpenTelemetry SDK → exporter. Do not assume `tracing-opentelemetry` exports the OpenTelemetry Logs signal; verify the configured Logs bridge.
- Filters run before export. Disabled spans/events cannot be recovered downstream, and a `TRACE` filter changes exported volume rather than only console verbosity.
- Treat `otel.name`, `otel.kind`, `otel.status_code`, and `otel.status_description` as reserved. Use semantic-convention attributes and bounded cardinality; avoid raw payloads and secrets.
- Event levels are exported; span level fields depend on the layer configuration. Error-level events can mark a span as failed, so preserve error semantics while changing volume controls.

## Review Workflow

1. Enumerate records, spans, fields, filters, exporters, and their call boundaries.
2. For each record, state the operational question, semantic severity, volume treatment, context, and privacy risk.
3. Classify the smallest correction: delete, add fields, restructure as a span, change level, filter/sample, aggregate as a metric, or redact.
4. Review retries by outcome and inspect repeated clusters separately from the first failure.
5. Report current → recommended output, the reason, and the narrowest validation or captured-subscriber test.

## Boundary With Related Skills

- Use `rust-tracing-context` for async span propagation and task correlation across `tokio::spawn`, `.await`, and task-local scopes.
- Use `rust-async-concurrency` for task ownership, cancellation, synchronization, executor health, and backpressure.
- Use `rust-error-silence` for ignored errors, panic boundaries, and propagation; this skill owns duplicate logging only as it affects emitted records.
- Use `rust-architecture-entropy-review` for cross-module logging sources of truth or divergent execution routes.

## Official Basis

- [`tracing::Level`](https://docs.rs/tracing/latest/tracing/struct.Level.html)
- [`tracing` core concepts](https://docs.rs/tracing/latest/tracing/index.html#core-concepts)
- [`tracing-opentelemetry`](https://docs.rs/tracing-opentelemetry/latest/tracing_opentelemetry/)
- [`opentelemetry-otlp`](https://docs.rs/opentelemetry-otlp/latest/opentelemetry_otlp/)
