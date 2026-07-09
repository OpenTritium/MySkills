---
name: high-snr-log
description: Use when reviewing or writing log statements — eliminating noise logs, breadcrumb spam, empty cheerleading, code-translating logs, fat object dumping. Keywords: log, logger, logging, print, info, warn, error, debug, trace, SNR, signal-to-noise, log payload, log message, 日志
---

# High-SNR Log Reviewer

## Overview
Eliminate noise logs. Code is the best comment — logs must NOT translate code. Only log runtime unknowns (dynamic context like user IDs, latency, external error codes). Every log must pass the "So What?" test: can an operator act on it immediately?

## When to Use
- Reviewing PRs with log statements
- Refactoring log content/payload quality
- Auditing production log signal-to-noise ratio

## Rules Engine

1. **Anti-Echo** — Delete logs that merely restate the method name or code logic. Normal paths stay silent; log only on failure or anomaly.
2. **Context-Driven** — Every log MUST include entity IDs, trace IDs, state transitions. "User updated successfully" without a user ID is useless.
3. **No Narrative** — Kill prose/essays in logs. Replace with structured key-value tags: `logger.warn("Payment retry", orderId=88, attempt=2, reason="Timeout")`.
4. **Exception Precision** — In catch blocks, log the business inputs that caused the failure, not just the exception message. Never log AND rethrow (dual printing).

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `logger.debug("Enter function A")` / `logger.debug("Leave function A")` | Delete — use tracing/profiling tools instead |
| `logger.info("Save to DB success!")` | Delete or demote to TRACE — failure throws exception |
| `logger.debug("Request: {}", json.dumps(hugeRequest))` | Log only business ID and changed fields — avoid I/O pressure and PII leaks |

## Workflow
1. Identify logs that merely translate code — delete them
2. Find logs missing critical dynamic variables (IDs, latency, status values) — add them
3. Strip prose descriptions, convert to structured key-value style
4. Output optimized code with justifications for deletions
