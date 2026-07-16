---
name: unsafe-checker
description: 'Review Rust unsafe code and FFI for memory safety and ABI soundness. Use when code contains unsafe, raw pointers, extern functions, transmute, unions, repr(C), MaybeUninit, NonNull, or a SAFETY comment. 中文触发：不安全代码、FFI、裸指针、未定义行为、内存安全'
---

# Unsafe Rust Reviewer

## Core Rules

Treat every `unsafe` block, `unsafe fn`, and FFI boundary as a proof obligation.

1. **Pointer validity** — Prove provenance, alignment, initialization, lifetime, and bounds for every dereference.
2. **Aliasing** — Preserve Rust's exclusive-mutation and shared-read rules; justify raw-pointer reborrows and `Send`/`Sync` implementations.
3. **Layout and initialization** — Verify `repr(C)`, field order, padding, ownership, drop behavior, `MaybeUninit`, unions, and transmute size/alignment.
4. **FFI boundary** — Match ABI, calling convention, nullability, ownership, string encoding, thread-safety, and unwind behavior on both sides.
5. **Safe surface** — Keep unsafe code small; expose a safe wrapper only when all caller-observable preconditions are enforced.

## Review Process

1. State the invariant required before each unsafe operation.
2. Identify who establishes, preserves, and invalidates that invariant.
3. Check error, panic, cancellation, and early-return paths for leaks or invalid cleanup.
4. Check `SAFETY` text against the actual proof; it must explain why the operation is sound, not restate the code.
5. Report definite unsoundness separately from portability or maintainability concerns.

## Common Mistakes

| Finding | Correction |
|---|---|
| Dereference justified only by `!is_null()` | Also prove alignment, initialization, provenance, and lifetime |
| `transmute` used for parsing or layout conversion | Use a checked conversion or an explicit `repr(C)`/byte-level operation |
| `extern "C"` wrapper ignores ownership or nullability | Document and enforce both sides of the contract |
| `unsafe impl Send/Sync` has no invariant proof | Prove thread-safety or remove the impl |
| FFI callback can unwind into foreign code | Catch or prohibit unwinding at the boundary |
| Full function marked `unsafe` for one operation | Isolate the smallest unsafe block and validate inputs first |
