---
name: error-silence
description: Use when reviewing error handling — swallowed errors, empty catch blocks, ignored return values, unwrap/expect in production, lost error context, dual printing, panic vs error return. Keywords: error handling, exception, catch, throw, unwrap, expect, panic, Result, ?, try-catch, error context, error wrapping, ignore error, silence, 错误处理, 异常, 吞异常
---

# Error Silence Reviewer

## Overview
The worst error is the one you never see. Silent failures, lost context, and panics that take down production are not edge cases — they are code review defects. Every error MUST be either handled, propagated with context, or explicitly documented as ignorable.

## When to Use
- Empty catch blocks or `_ = err` in code
- `.unwrap()` / `.expect()` in non-startup, non-test code
- Logging AND rethrowing the same error (dual printing)
- Generic error wrapping that discards the original error type
- `panic()` calls in library or long-running service code

## Rules Engine

1. **Never Swallow** — Empty `catch {}` or `_ = err` must be justified or deleted. If an error is truly ignorable, comment WHY with a specific scenario, not "this should never fail".

2. **Context on Every Hop** — Every error propagation boundary MUST wrap with context: what operation failed, on what input. `return err` loses the trail; `return fmt.Errorf("parsing order %s: %w", orderID, err)` preserves it.

3. **No Dual Printing** — Logging AND returning/rethrowing the same error creates duplicate stack traces. Log at the outermost boundary ONLY; intermediate layers wrap and return.

4. **Unwrap is a Crash** — `.unwrap()` / `.expect()` in production code is a deferred panic. Replace with proper error propagation or a fallback. Allowed ONLY in: one-shot `main()`, test helpers, or when a prior invariant guarantee makes failure truly impossible (with a comment proving it).

5. **Panic Boundaries** — Library code must NEVER panic. Long-running services panic only on unrecoverable startup failures (missing config, port bind). Everything else returns an error.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `catch(e) { }` / `except: pass` / `_ = err` | Handle it, or comment WHY it's safe with specific scenario |
| `result.unwrap()` in handler logic | `result?` with proper error propagation chain |
| `log.Error("order failed", err); return err` | Return wrapped error only; let top-level handler log |
| `errors.New("failed")` discards underlying cause | `fmt.Errorf("syncing user %s: %w", uid, err)` preserves wrapped error |
| `panic("should never happen")` deep in library | Return error; let caller decide if it's fatal |
| `if err != nil { return err }` with zero context | Wrap: `fmt.Errorf("reading config %s: %w", path, err)` |
| `_ = file.Close()` ignoring close error | Handle or annotate: `defer func() { _ = f.Close() }` with comment why write-only fd is safe |
| `try { ... } catch (Exception e) { ... }` catching too broadly | Catch specific exception types; broad catch at outermost boundary only |

## Workflow
1. Find every `.unwrap()`, `panic()`, `except:`, `catch {}` — each is a candidate defect
2. Trace error propagation paths: is context added at each boundary? Or lost?
3. Identify dual-printing (log + return/throw same error) — remove the intermediate log
4. For each ignored error, demand a comment explaining WHY it's safe, or add handling
5. Output fixes with before/after and the specific context each wrapped error now carries
