---
name: log-level
description: Use when reviewing log levels (ERROR/WARN/INFO/DEBUG/TRACE) in code — enforcing correct severity, preventing production log flooding, retry spamming, routine success logging. Keywords: log level, ERROR, WARN, INFO, DEBUG, TRACE, log severity, logging level, tracing, production logging, 日志级别, 日志分级
---

# Log Level Reviewer

## Overview
INFO+ must never flood production. Production logs = critical state changes and anomalies only; high-frequency/routine I/O belongs at DEBUG/TRACE. Level is an operator contract: ERROR pages someone, WARN surfaces a degradation worth scanning, INFO marks a lifecycle milestone, DEBUG/TRACE are for a fire.

## When to Use
- Setting/reviewing log levels (`error!`/`warn!`/`info!`/`debug!`/`trace!`)
- Production log noise investigation
- Auditing ERROR/WARN/INFO severity semantics

## Decision Tree

Apply in priority order:

1. **Frequency** — Hot loop, high-frequency path, or periodic trigger? → DEBUG/TRACE. (Exception: first failure in a periodic task = WARN.)
2. **Fatal** — Halts process/transaction, needs manual intervention (restart, config, downstream fix)? → ERROR.
3. **Self-healing** — Unexpected failure with auto retry/fallback, system serves degraded? → WARN. First failure only; retries = DEBUG, exhaustion = ERROR.
4. **State transition** — Low-frequency lifecycle (startup/shutdown, election, pool init, batch done)? → INFO.
5. **Otherwise** → DEBUG.

> WARN vs ERROR: WARN = degraded but alive (self-healed/retrying); ERROR = needs a human. Retried timeout that succeeds = WARN; same timeout exhausting retries and dropping = ERROR.

## Level Reference

| Level | Semantic | Example |
|---|---|---|
| ERROR | Needs a human; cannot continue | Init failure, persistent disconnect, retry exhaustion, data corruption |
| WARN | Degraded but alive; scanned not paged | First I/O timeout (retries), breaker trip, missing non-critical config, rate limit |
| INFO | Low-frequency milestone (lifecycle, not per-request) | Listening, batch start/end, pool initialized, hot-reload |
| DEBUG | Troubleshooting context | Per-request latency/size, retry internals, state-sync params |
| TRACE | Micro-events | Packet encode/decode, per-row dispatch, heartbeat, iterator step |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `info!` on every DB write / HTTP 200 | DEBUG — success is default, not a milestone |
| `warn!` on every retry in a loop | WARN first only, DEBUG retries, ERROR exhaustion |
| `warn!` on expected stream end / channel close / ctx cancel | DEBUG — normal lifecycle, not anomaly |
| `error!` on a retried failure that succeeds | WARN — recovered; ERROR is for needing a human |
| `info!("request received")` per request | `tracing` span — fields carry IDs, not the level |

## Workflow
1. Classify each log by the Decision Tree (frequency → fatal → self-healing → state → else)
2. ERROR genuinely needs a human? Downgrade recovered failures to WARN
3. WARN is first-failure or real degradation? Routine retries → DEBUG
4. INFO is low-frequency lifecycle, not per-request/per-row? Demote the rest
5. Level-inverted pairs: retried failure logged ERROR that succeeds → WARN; exhaustion logged WARN → ERROR
6. Output old level → new level + one-line Decision Tree reason
