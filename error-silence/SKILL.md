---
name: error-silence
description: Use when reviewing Rust error handling — swallowed errors, ignored Results, unwrap/expect in production, lost error context, missing .context(), panic vs Result, anyhow vs thiserror. Keywords: error handling, Result, Option, ?, unwrap, expect, panic, anyhow, thiserror, context, .ok(), let _ =, error propagation, error wrapping, ignore error, silence, 错误处理, 吞异常
---

# Error Silence Reviewer

## Overview
The worst error is the one you never see. Silent failures, lost context, and panics that take down production are not edge cases — they are code review defects. In Rust every fallible operation returns a `Result`; discarding it is a deliberate choice that must be justified. Every error MUST be either handled, propagated with context, or explicitly documented as ignorable.

## When to Use
- `let _ = result` or `.ok()` that drops an `Err`
- `.unwrap()` / `.expect()` / `.panic!()` in non-startup, non-test code
- `?` propagation with no context added across a boundary
- Generic error wrapping that discards the original error type or its downcast target
- Library code that panics instead of returning `Result`

## Rules Engine

1. **Never Swallow** — `let _ = tx.send()`, `.ok()` dropping `Err`, `if let Ok(_) = ...` that ignores the error arm: each must be justified or handled. If an error is truly ignorable, comment WHY with a specific scenario ("receiver dropped: shutdown in progress, the send failing is the signal"), never "this should never fail".

2. **Context on Every Hop** — Every `?` that crosses a boundary (function, module, layer) should carry context. Bare `?` loses the trail; `result.context("parsing order {order_id}")?` (anyhow) or a typed enum variant (thiserror) preserves it. Ask: if this error surfaces in a log at 3am, can you trace which operation on which input failed?

3. **No Dual Printing** — Logging an error AND propagating it (`tracing::error!(?e); return Err(e)`) creates duplicate stack traces and double-counted alerts. Log at the outermost boundary ONLY; intermediate layers wrap and return with `?`. (The full log-noise treatment lives in the `high-snr-log` skill.)

4. **Unwrap is a Crash** — `.unwrap()` / `.expect()` / direct indexing `v[i]` in production code is a deferred panic. Replace with `?`, `.unwrap_or_else()`, or `match`. Allowed ONLY in: one-shot `main()`, `#[cfg(test)]`, or when a prior invariant guarantee makes failure truly impossible (with a comment proving it — and even then prefer `.expect("reason")` over `.unwrap()` so the panic names the assumption).

5. **Panic Boundaries** — Library `pub fn` must NEVER panic on any input; return `Result` for fallible operations and document panics only for truly impossible states. Long-running services panic only on unrecoverable startup failures (missing config, port bind); everything else is a returned error. A panic in a request handler takes down the whole worker, not just the request.

6. **Choose the Right Tool** — `anyhow::Result` for applications where you propagate opaque errors to a single boundary; `thiserror` + a typed error enum for libraries and APIs where callers `match` on variants. Mixing both: define typed errors at module boundaries, convert to `anyhow::Error` with `.context()` as you propagate outward.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `let _ = tx.send(res);` | Handle or annotate: `if tx.send(res).is_err() { /* receiver gone, shutting down */ }` |
| `.ok()` dropping a meaningful `Err` | Keep `Result` and propagate with `?`, or `match` and handle the error arm |
| `config.parse().unwrap()` in handler logic | `config.parse().context("parsing config")?` |
| `return Err(e)` with zero context across a boundary | `Err(e).context("syncing user {user_id}")?` or a typed variant with the input |
| `tracing::error!(?e); return Err(e)` | Return wrapped error only; let the boundary log once (dual printing) |
| `errors::Generic("failed")` discards the cause | `#[error("syncing user {user_id}: {source}")]` keeps the source chain |
| `.unwrap()` on a `Mutex::lock()` in a request path | `match lock() { Ok(g) => ..., Err(PoisonError) => ... }` — decide explicitly |
| `panic!("should never happen")` deep in a library | Return `Result`; let the caller decide if it's fatal |

## Workflow
1. Find every `let _ =`, `.ok()`, `.unwrap()`, `.expect()`, `panic!`, bare `[i]` — each is a candidate defect
2. Trace each `?`: is context added when crossing a boundary? If not, add `.context()` or a typed variant
3. Identify dual-printing sites (log + return the same error) — remove the intermediate log
4. For each swallowed error, demand a comment with a specific scenario, or add handling
5. Verify library `pub fn`s return `Result` instead of panicking; verify services only panic on startup
6. Output fixes with before/after and the specific context each wrapped error now carries
