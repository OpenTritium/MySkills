---
name: error-silence
description: 'Review Rust error handling: ignored outcomes, unwrap/expect panic sites, lost context, propagation boundaries, anyhow/thiserror choices, and duplicate logging. Keywords: error handling, Result, Option, ?, unwrap, expect, panic, anyhow, thiserror, context, .ok(), let _ =, error propagation, error wrapping, ignore error, silence, 错误处理, 吞异常'
---

# Error Silence Reviewer

## Overview
Silent failures and lost context are review defects. Handle every fallible outcome deliberately: propagate it, recover it, convert it, or document why it is ignorable. `Result`, `Option`, and documented panics are different API contracts.

## Rules Engine

1. **Do Not Swallow** — `let _ = tx.send()`, `.ok()`, `if let Ok(_) = ...`: each must be justified or handled. If ignorable, record the specific scenario ("receiver dropped: shutdown in progress"), never "should never fail".

2. **Context at Boundaries** — When `?` crosses an operation or domain boundary and the source is ambiguous, add `.context("parsing order {order_id}")?` (anyhow) or a typed variant (thiserror). Test: at 3am, can you trace which op on which input failed?

3. **Avoid Dual Printing** — Logging AND propagating the same error (`error!(?e); return Err(e)`) often duplicates traces and alerts. Prefer logging at the handling boundary; intermediate layers add context and return unless they add distinct actionable information. (Full treatment: `high-snr-log`.)

4. **Audit Panic Sites** — `.unwrap()` / `.expect()` / `v[i]` can panic. Use `?`, `.unwrap_or_else()`, or `match` for ordinary runtime failures; keep direct operations only when a local invariant or documented precondition makes the panic intentional, and prefer `.expect("reason")` over `.unwrap()`.

5. **Panic Boundaries** — Public APIs should return errors for ordinary invalid input and document intentional panics. Services may panic for unrecoverable initialization or violated internal invariants; keep request-path panics out of expected failures.

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
| `panic!("should never happen")` on caller-controlled input | Return an error or document the precondition; reserve panic for a proven internal invariant |

## Workflow
1. Find every `let _ =`, `.ok()`, `.unwrap()`, `.expect()`, `panic!`, bare `[i]` — candidate defects
2. Trace each `?`: context added at boundaries? If not, `.context()` or typed variant
3. Dual-printing sites (log + return same error) — remove the intermediate log
4. Each swallowed error: demand a specific-scenario comment, or add handling
5. Verify public APIs handle ordinary invalid input; verify intentional panics have a documented boundary
6. Output before/after with the context each wrapped error now carries
