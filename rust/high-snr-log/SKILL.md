---
name: high-snr-log
description: 'Review Rust log statements and structured `tracing` records for noise, breadcrumb spam, empty success messages, code-translating text, fat payloads, and missing context. Keywords: log, logger, logging, print, info, warn, error, debug, trace, tracing, log::, env_logger, structured logging, SNR, signal-to-noise, log payload, log message, 日志'
---

# High-SNR Log Reviewer

## Overview
Logs cost disk, ingest, retention, and operator scan time — each line should answer an operational question. Prefer runtime context that distinguishes an entity, state, latency, or outcome; static lifecycle messages can still be useful. Routine success usually stays silent.

## Rules Engine

1. **Anti-Echo** — Delete logs restating the method name or control flow. `fn save() { info!("saving..."); }` adds little beyond the span. Keep normal paths quiet unless the milestone is operationally useful; log failures, anomalies, or decisions.

2. **Context-Driven** — Every log carries dynamic values an operator needs: entity IDs, trace IDs, state transitions, counts, latencies. `info!(user_id, "login")` has signal; `info!("login success")` has none. Can't name the question it answers → delete.

3. **Structured First** — Prefer indexable fields over interpolated `format!` text: `warn!(order_id, attempt, reason = %e, "payment retry failed")`. Keep a concise human-readable message when it helps operators.

4. **No Dual Printing** — Never log AND propagate an error (`error!(?e); return Err(e)`); the outer boundary logs once. (Full rule: `error-silence`.)

5. **Fat-Object Discipline** — Avoid whole structs/bodies/JSON by default at any level: they bloat ingest and may leak secrets or PII. Emit bounded, redacted fields; TRACE is not a privacy boundary.

6. **Span, Not Breadcrumb** — "entering/leaving function X" → `#[tracing::instrument]` span or `info_span!`, not manual `debug!("enter X")`. Spans give timing, nesting, correlation; breadcrumbs give noise.

## Signal Logs (keep)

| Type | Example |
|---|---|
| Failure with context | `error!(order_id, attempt, "gateway timeout after 3 retries")` |
| State transition | `info!(node_id, peers = cluster.len(), "elected leader")` |
| Recoverable anomaly | `warn!(user_id, "rate limit hit, backing off 60s")` |
| Decision + reason | `info!(request_id, "routed to fallback: primary unhealthy")` |
| Performance | `warn!(endpoint, p99_latency_ms, "breached SLO")` |

## Noise Logs (delete)

| Anti-Pattern | Fix |
|---|---|
| `debug!("entering A")` / `debug!("leaving A")` | `#[tracing::instrument]` span — timing + correlation free |
| `info!("save to DB success!")` | Delete — log failure/anomaly only |
| `info!("processing request")` at handler top | `#[tracing::instrument(skip(req))]` — fields carry IDs |
| `info!("request: {:?}", request)` | Log bounded identifying fields only: `info!(request_id, user_id, path = %req.path)` |
| `error!(?e, "failed"); return Err(e)` | Delete the log; boundary logs once |
| `info!("done")` cheerleading | Delete — unless slow batch completion, then add duration |
| `println!("{:?}", body)` debug leftover | Delete or emit bounded, redacted fields under an explicit diagnostic filter |

## Workflow
1. Enumerate every log; each must justify its existence
2. "So What?" test — name the operational question it answers. No answer → delete
3. Delete anti-echo logs: success cheers, enter/leave breadcrumbs → spans
4. Add missing context (entity ID, count, latency) to survivors
5. Convert freeform `format!` → structured fields; fat dumps → `trace!` or delete
6. No dual printing: one boundary logs, rest propagate with `?`
7. Output per change: deleted (why), modified (context added), or converted to span
