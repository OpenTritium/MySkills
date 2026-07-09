---
name: log-level
description: Use when reviewing log levels (ERROR/WARN/INFO/DEBUG/TRACE) in code — enforcing correct severity, preventing production log flooding, retry spamming, routine success logging. Keywords: log level, ERROR, WARN, INFO, DEBUG, TRACE, log severity, logging level, tracing, production logging, 日志级别, 日志分级
---

# Log Level Reviewer

## Overview
INFO and above must never flood production logs. Production logs reflect only critical state changes and anomalies. High-frequency data and routine I/O belong at DEBUG/TRACE. The right level is an operator contract: ERROR pages someone, WARN surfaces a degradation worth scanning, INFO marks a lifecycle milestone, DEBUG/TRACE are for you during a fire.

## When to Use
- Setting or reviewing log levels (`error!`/`warn!`/`info!`/`debug!`/`trace!`) in code
- Production log noise investigation
- Auditing that ERROR/WARN/INFO usage follows severity semantics

## Decision Tree

Apply in priority order for every log statement:

1. **Frequency Check** — Is this in a hot loop, high-frequency request path, or periodic trigger? → Demote to DEBUG/TRACE. (Exception: first failure in a periodic task can be WARN.)
2. **Fatal Check** — Does this event halt the process/transaction and require manual intervention (restart, config change, fix downstream)? → ERROR.
3. **Self-healing Check** — Is this an unexpected failure with automatic retry/fallback/recovery, where the system continues to serve but degraded? → WARN. Only the *first* failure is WARN; subsequent retries are DEBUG, and final exhaustion (gave up) is ERROR.
4. **State Transition Check** — Is this a low-frequency lifecycle event (startup/shutdown, leader election, connection pool initialized, long batch completed)? → INFO.
5. **Otherwise** → DEBUG.

> Note the WARN vs ERROR boundary: WARN = degraded but alive (system self-healed or is retrying); ERROR = needs a human. A retried timeout that eventually succeeds is WARN; the same timeout that exhausts retries and drops the request is ERROR.

## Level Reference

| Level | Semantic | Example |
|---|---|---|
| ERROR | Needs a human; process/transaction cannot continue | Core resource init failure, persistent dependency disconnection, retry exhaustion, data corruption |
| WARN | Degraded but alive; anomaly scanned, not paged | First I/O timeout (will retry), circuit breaker trip, missing non-critical config, rate limit hit |
| INFO | Low-frequency milestone (lifecycle, not per-request) | Server listening, batch job start/end, connection pool initialized, config hot-reload success |
| DEBUG | Troubleshooting context during a fire | Per-request latency/size, retry internals, state sync parameters, resource cleanup |
| TRACE | Micro-event tracing | Packet encode/decode, per-row dispatch, heartbeat packets, iterator step |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `info!` on every DB write success / HTTP 200 | Demote to DEBUG — success is the default, not a milestone |
| `warn!` on every retry attempt in a loop | WARN on first failure only, DEBUG for subsequent retries, ERROR on exhaustion |
| `warn!` on expected stream end, channel close, context cancellation | Demote to DEBUG — these are normal lifecycle, not anomalies |
| `error!` on a retried failure that succeeds | WARN — the system recovered; ERROR reserves it for needing a human |
| `info!("request received")` on every request | Use a `tracing` span, not a log — fields carry the IDs, not the level |

## Workflow
1. List every log statement in scope; classify each by the Decision Tree (frequency → fatal → self-healing → state → else)
2. Verify each ERROR genuinely needs a human; downgrade recovered failures to WARN
3. Verify each WARN is either a first-failure or a real degradation — convert routine retries to DEBUG
4. Verify each INFO is low-frequency lifecycle, not per-request/per-row — demote the rest
5. Spot-check for level-inverted pairs: a retried failure logged ERROR that succeeds → WARN; a final exhaustion logged WARN → ERROR
6. Output each log with old level → new level and the one-line reason from the Decision Tree

