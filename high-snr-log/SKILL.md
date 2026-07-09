---
name: high-snr-log
description: Use when reviewing or writing log statements — eliminating noise logs, breadcrumb spam, empty cheerleading, code-translating logs, fat object dumping. Keywords: log, logger, logging, print, info, warn, error, debug, trace, tracing, log::, env_logger, structured logging, SNR, signal-to-noise, log payload, log message, 日志
---

# High-SNR Log Reviewer

## Overview
Logs cost disk, ingest, retention, and operator scan time — every line must earn its keep. A log must carry runtime info the code can't know statically (dynamic IDs, latencies, external states, decision outcomes). If you can predict the message from source, it's noise. Routine success is not a log.

## When to Use
- PRs adding/modifying log statements
- Refactoring log content/payload quality
- Auditing production log volume (high cost, low signal)
- Migrating to structured logging (`tracing` spans/fields)

## Rules Engine

1. **Anti-Echo** — Delete logs restating the method name or control flow. `fn save() { info!("saving..."); }` is zero value — the span says "save". Normal paths stay silent; log failure, anomaly, or decision points only.

2. **Context-Driven** — Every log carries dynamic values an operator needs: entity IDs, trace IDs, state transitions, counts, latencies. `info!(user_id, "login")` has signal; `info!("login success")` has none. Can't name the question it answers → delete.

3. **Structured, Not Prose** — Kill freeform `format!` messages. Use indexable fields: `warn!(order_id, attempt, reason = %e, "payment retry failed")`. Grepping `order_id=88 AND level>=warn` works; grepping a sentence doesn't.

4. **No Dual Printing** — Never log AND propagate an error (`error!(?e); return Err(e)`); the outer boundary logs once. (Full rule: `error-silence`.)

5. **Fat-Object Discipline** — Never dump whole structs/bodies/JSON at INFO/WARN — bloats ingest, leaks PII, buries the one field that matters. Log 1–3 identifying fields; demote full dumps to `trace!` behind a flag, or drop.

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
| `info!("request: {:?}", request)` | Log identifying fields only: `info!(request_id, user_id, path = %req.path)` |
| `error!(?e, "failed"); return Err(e)` | Delete the log; boundary logs once |
| `info!("done")` cheerleading | Delete — unless slow batch completion, then add duration |
| `println!("{:?}", body)` debug leftover | Delete or demote to `trace!` behind a flag |

## Workflow
1. Enumerate every log; each must justify its existence
2. "So What?" test — name the operational question it answers. No answer → delete
3. Delete anti-echo logs: success cheers, enter/leave breadcrumbs → spans
4. Add missing context (entity ID, count, latency) to survivors
5. Convert freeform `format!` → structured fields; fat dumps → `trace!` or delete
6. No dual printing: one boundary logs, rest propagate with `?`
7. Output per change: deleted (why), modified (context added), or converted to span
