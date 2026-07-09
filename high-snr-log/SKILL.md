---
name: high-snr-log
description: Use when reviewing or writing log statements — eliminating noise logs, breadcrumb spam, empty cheerleading, code-translating logs, fat object dumping. Keywords: log, logger, logging, print, info, warn, error, debug, trace, tracing, log::, env_logger, structured logging, SNR, signal-to-noise, log payload, log message, 日志
---

# High-SNR Log Reviewer

## Overview
Logs cost money and attention at scale — disk, ingest, retention, and the operator's scan time. Every line must earn its keep. A log must carry runtime information the code cannot know statically (dynamic IDs, latencies, external states, decision outcomes). If you can predict the message from reading the source, it's noise. Routine success is not a log; the next person's panic is.

## When to Use
- Reviewing PRs that add or modify log statements
- Refactoring log content / payload quality
- Auditing production log volume — high ingest cost, low signal
- Migrating to structured logging (`tracing` spans/fields)

## Rules Engine

1. **Anti-Echo** — Delete logs that merely restate the method name or the control flow. `fn save() { info!("saving..."); ... }` is zero value — the span already says "save". Normal paths stay silent; log only on failure, anomaly, or a decision point.

2. **Context-Driven** — Every log MUST carry the dynamic values an operator needs to act: entity IDs, trace IDs, state transitions, counts, latencies. `info!(user_id, "login")` carries signal; `info!("login success")` carries none. If you can't name the question this log answers, delete it.

3. **Structured, Not Prose** — Kill freeform `format!`-style messages. Use structured fields the logging backend can index and filter: `warn!(order_id, attempt, reason = %e, "payment retry failed")`. An operator grepping `order_id=88 AND level>=warn` gets value; grepping a sentence does not.

4. **No Dual Printing** — Never log an error AND propagate it (`tracing::error!(?e); return Err(e)`); the outer boundary should log it once. See the `error-silence` skill for the full rule on error/propagation overlap.

5. **Fat-Object Discipline** — Never dump whole structs, request bodies, or JSON blobs at INFO/WARN. They bloat ingest, leak PII, and bury the one field that matters. Log the 1–3 business fields that identify the event; demote full dumps to `trace!` behind a feature flag, or drop them.

6. **Span, Not Breadcrumb** — For "entering / leaving function X" logs, use a `tracing` span (`#[tracing::instrument]` or `info_span!`) instead of manual `debug!("enter X")` / `debug!("exit X")`. Spans give you timing, nesting, and correlation for free; breadcrumbs give you noise.

## Signal Logs (keep)

| Type | Example |
|---|---|
| Failure with identifying context | `error!(order_id, attempt, "payment gateway timeout after 3 retries")` |
| Key state transition | `info!(node_id, peers = cluster.len(), "elected leader")` |
| Anomalous-but-recoverable | `warn!(user_id, "rate limit hit, backing off 60s")` |
| Decision with reason | `info!(request_id, "routed to fallback region: primary unhealthy")` |
| Performance signal | `warn!(endpoint, p99_latency_ms, "breached SLO")` |

## Noise Logs (delete)

| Anti-Pattern | Fix |
|---|---|
| `debug!("entering function A")` / `debug!("leaving A")` | Replace with `#[tracing::instrument]` span — timing + correlation for free |
| `info!("save to DB success!")` | Delete — success is the default; log only on failure or anomaly |
| `info!("processing request")` at handler top | Use `#[tracing::instrument(skip(req))]` — fields carry the IDs, not the message |
| `info!("request: {:?}", request)` dumping a fat struct | Log identifying fields only: `info!(request_id, user_id, path = %req.path)` |
| `error!(?e, "failed"); return Err(e)` | Delete the log; let the outer boundary log once (dual printing) |
| `info!("done")` / `info!("complete")` cheerleading | Delete — unless a slow batch completion, in which case add duration |
| `println!("{}", json.dumps(body))` debug leftover | Delete or demote to `trace!` behind a flag; never in production path |

## Workflow
1. Enumerate every log statement in the diff (or target module); each must justify its existence
2. For each, apply the "So What?" test — name the operational question it answers. No answer → delete.
3. Delete anti-echo logs: success-path cheers, enter/leave breadcrumbs (convert to spans), restating-the-method-name
4. Check every surviving log for identifying context — add the entity ID, count, or latency it's missing
5. Convert freeform `format!` messages to structured fields; move fat-object dumps to `trace!` or delete
6. Verify no dual printing: an error should be logged at exactly one boundary, propagated elsewhere with `?`
7. Output the cleaned code with, per change: deleted (why), modified (what context added), or converted to span
