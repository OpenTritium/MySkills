---
name: log-level
description: Use when reviewing log levels (ERROR/WARN/INFO/DEBUG/TRACE) in code — enforcing correct severity, preventing production log flooding, retry spamming, routine success logging. Keywords: log level, ERROR, WARN, INFO, DEBUG, TRACE, log severity, logging level, production logging, 日志级别, 日志分级
---

# Log Level Reviewer

## Overview
INFO and above must never flood production logs. Production logs reflect only critical state changes and anomalies. High-frequency data and routine I/O belong at DEBUG/TRACE.

## When to Use
- Setting or reviewing log levels in code
- Production log noise investigation
- Auditing that ERROR/WARN/INFO usage follows severity semantics

## Decision Tree

Apply in priority order for every log statement:

1. **Frequency Check** — Is this in a hot loop, high-frequency request path, or periodic trigger? → Demote to DEBUG/TRACE. (Exception: first failure in a periodic task can be WARN.)
2. **Fatal Check** — Does this event halt the process/transaction and require manual intervention (restart, config change, fix downstream)? → ERROR.
3. **State Transition Check** — Is this a key lifecycle event (startup/shutdown, failover, connection established, long batch completed)? → INFO.
4. **Self-healing Check** — Is this an unexpected failure with automatic retry/fallback/recovery, no manual action needed? → WARN. (Only first failure is WARN; subsequent retries are DEBUG.)
5. **Otherwise** → DEBUG.

## Level Reference

| Level | Semantic | Example |
|---|---|---|
| ERROR | Fatal, manual action required | Core resource init failure, persistent dependency disconnection, data corruption causing panic |
| WARN | Recoverable anomaly | Single I/O timeout (will retry), circuit breaker trip, missing non-critical config, rate limit hit |
| INFO | Low-frequency milestone | Server listening, batch job start/end, connection pool initialized, config hot-reload success |
| DEBUG | Troubleshooting context | Per-request latency/size, retry internals, state sync parameters, resource cleanup |
| TRACE | Micro-event tracing | Packet encode/decode, per-row dispatch, heartbeat packets, pointer movement inside loops |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `INFO` on every DB write success / HTTP 200 | Demote to DEBUG |
| `WARN` on every retry attempt in a loop | WARN on first failure only, DEBUG for subsequent retries |
| `WARN` on expected stream end, channel close, context cancellation | Demote to DEBUG — normal lifecycle |
