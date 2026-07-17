---
name: log-level
description: 'Review and correct ERROR/WARN/INFO/DEBUG/TRACE levels in Rust `tracing`, `tracing-opentelemetry`, and logging code. Use when choosing log or span levels, distinguishing DEBUG from TRACE, auditing retry/noise severity, preventing high-volume telemetry, or separating semantic priority from event frequency. Keywords: log level, ERROR, WARN, INFO, DEBUG, TRACE, verbosity, severity, tracing, tracing-opentelemetry, OTLP, OpenTelemetry, production logging, 日志级别, 日志分级'
---

# Log Level Reviewer

## Core Rules

- Treat a level as the verbosity and priority of a structured span or event, not as message length or field count.
- Use the official order: `ERROR < WARN < INFO < DEBUG < TRACE`. A `DEBUG` filter includes `ERROR`–`DEBUG`; `TRACE` includes all five.
- Select level by significance first; control volume with sampling, aggregation, rate limiting, metrics, or removal.
- Use a span for a period of work and context; use an event for a moment within that span.
- Keep review and edit scope proportional to the finding. A broad level change needs a semantic reason at each call site; do not mass-rewrite events solely to reduce warning volume when filtering, sampling, aggregation, or metrics would address the operational cost.

| Level | Use for |
|---|---|
| `ERROR` | Very serious failure: broken contract, lost work, corruption, or invariant violation |
| `WARN` | Hazard or degradation that still permits progress |
| `INFO` | Useful normal information, lifecycle, or externally meaningful state change |
| `DEBUG` | Lower-priority diagnostic state explaining decisions or recovery |
| `TRACE` | Very-low-priority, often extremely verbose execution-flow evidence |

`TRACE` is more verbose than `DEBUG`, not necessarily more detailed. Use `DEBUG` for a useful troubleshooting boundary; use `TRACE` to reconstruct fine-grained control flow.

## tracing-opentelemetry / OTLP

`tracing` macros → subscriber/layers → `tracing-opentelemetry` → OpenTelemetry SDK → exporter such as `opentelemetry-otlp` → backend.

- Treat spans/events as trace telemetry, not automatically as log records. `tracing-opentelemetry` bridges traces and metrics; use a Logs-specific bridge for the OpenTelemetry Logs signal.
- Filters run before export: disabled spans/events cannot be recovered by OTLP. A `TRACE` filter changes exported trace-event volume, not just console verbosity.
- Treat `otel.name`, `otel.kind`, `otel.status_code`, and `otel.status_description` as reserved. Use semantic-convention attributes; avoid secrets, raw payloads, and unbounded-cardinality values.
- Event levels are exported; span level fields require the layer's level option. Error-level events may mark a span status as error, so do not downgrade `ERROR` merely to reduce noise.

## Review Process

1. Classify significance using the table; do not start with frequency.
2. Choose span, event, attribute, exception, status, metric, or local log.
3. Check filter boundary, export cost, cardinality, and sensitive data.
4. Review retries by outcome: expected attempts usually `DEBUG`, real degradation `WARN`, terminal impact by severity.
5. Keep repeated serious events serious; suppress repetition separately.
6. Prioritize semantically incorrect levels and high-value clusters; treat broad noise cleanup as a separate, explicitly scoped change.

## Common Mistakes

| Mistake | Correction |
|---|---|
| Hot-loop event automatically becomes `TRACE` | Preserve severity; control volume independently |
| `warn!` on every expected retry | Use `DEBUG`; reserve `WARN` for actual hazard/degradation |
| `info!` per request/item | Prefer a span, metric, `DEBUG`/`TRACE`, or no event |
| Downgrade `error!` to clean OTLP output | Preserve error/status semantics; filter, sample, or aggregate |
| Export full payload or error debug output | Emit bounded, redacted diagnostic fields |
| Treat `tracing-opentelemetry` as a log exporter | Use the appropriate OpenTelemetry Logs bridge |

## Review Output

For each finding, report the current → recommended level, significance/filter reason, volume treatment, and whether a span/field/metric is better. For OTLP, state any effect on exported span/event/status semantics.

## Official Basis

- [`tracing::Level`](https://docs.rs/tracing/latest/tracing/struct.Level.html)
- [`tracing` core concepts](https://docs.rs/tracing/latest/tracing/index.html#core-concepts)
- [`tracing-opentelemetry`](https://docs.rs/tracing-opentelemetry/latest/tracing_opentelemetry/)
- [`opentelemetry-otlp`](https://docs.rs/opentelemetry-otlp/latest/opentelemetry_otlp/)
