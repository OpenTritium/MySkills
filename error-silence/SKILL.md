---
name: error-silence
description: Use when reviewing Rust error handling — swallowed errors, ignored Results, unwrap/expect in production, lost error context, missing .context(), panic vs Result, anyhow vs thiserror. Keywords: error handling, Result, Option, ?, unwrap, expect, panic, anyhow, thiserror, context, .ok(), let _ =, error propagation, error wrapping, ignore error, silence, 错误处理, 吞异常
---

# Error Silence Reviewer

## Overview
The worst error is the one you never see. Silent failures, lost context, and panics in production are code review defects. Every fallible Rust op returns a `Result`; discarding it is a deliberate choice. Every error MUST be handled, propagated with context, or documented as ignorable.

## When to Use
- `let _ = result` or `.ok()` dropping an `Err`
- `.unwrap()` / `.expect()` / `panic!()` in non-startup, non-test code
- `?` with no context across a boundary
- Error wrapping that discards the original type / downcast target
- Library code that panics instead of returning `Result`

## Rules Engine

1. **Never Swallow** — `let _ = tx.send()`, `.ok()`, `if let Ok(_) = ...`: each must be justified or handled. If ignorable, comment WHY with a specific scenario ("receiver dropped: shutdown in progress"), never "should never fail".

2. **Context on Every Hop** — `?` crossing a boundary must carry context. Bare `?` loses the trail; `.context("parsing order {order_id}")?` (anyhow) or a typed variant (thiserror) preserves it. Test: at 3am, can you trace which op on which input failed?

3. **No Dual Printing** — Logging AND propagating an error (`error!(?e); return Err(e)`) duplicates traces and double-counts alerts. Log at the outermost boundary ONLY; intermediate layers wrap and return. (Full treatment: `high-snr-log`.)

4. **Unwrap is a Crash** — `.unwrap()` / `.expect()` / `v[i]` in production is a deferred panic. Use `?`, `.unwrap_or_else()`, or `match`. Allowed ONLY in: one-shot `main()`, `#[cfg(test)]`, or provably-impossible failure (with a comment — and prefer `.expect("reason")` over `.unwrap()`).

5. **Panic Boundaries** — Library `pub fn` must NEVER panic on any input; return `Result`. Services panic only on unrecoverable startup (missing config, port bind); everything else is a returned error. A handler panic kills the whole worker, not the request.

6. **Choose the Right Tool** — `anyhow::Result` for apps (opaque errors to one boundary); `thiserror` + typed enum for libraries/APIs (callers `match` variants). Mix: typed errors at boundaries, convert to `anyhow` with `.context()` outward.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `let _ = tx.send(res);` | `if tx.send(res).is_err() { /* receiver gone, shutting down */ }` |
| `.ok()` dropping a meaningful `Err` | Propagate with `?`, or `match` and handle |
| `config.parse().unwrap()` in handler | `config.parse().context("parsing config")?` |
| `return Err(e)` with no context across boundary | `Err(e).context("syncing user {user_id}")?` |
| `error!(?e); return Err(e)` | Return wrapped error only; boundary logs once |
| `errors::Generic("failed")` drops cause | `#[error("syncing user {user_id}: {source}")]` keeps chain |
| `.unwrap()` on `Mutex::lock()` in a request path | `match` — handle `PoisonError` explicitly |
| `panic!("should never happen")` in a library | Return `Result`; let the caller decide |

## Workflow
1. Find every `let _ =`, `.ok()`, `.unwrap()`, `.expect()`, `panic!`, bare `[i]` — candidate defects
2. Trace each `?`: context added at boundaries? If not, `.context()` or typed variant
3. Dual-printing sites (log + return same error) — remove the intermediate log
4. Each swallowed error: demand a specific-scenario comment, or add handling
5. Verify `pub fn`s return `Result`; verify services only panic at startup
6. Output before/after with the context each wrapped error now carries
