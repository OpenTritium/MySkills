---
name: unsafe-checker
description: 'Review Rust unsafe code and FFI for memory safety and ABI soundness, including test-only environment mutation, unsafe Send/Sync implementations, and low-unsafe codebase audits. Use when code contains unsafe, raw pointers, extern functions, transmute, unions, repr(C), MaybeUninit, NonNull, set_var/remove_var, or a SAFETY comment. 中文触发：不安全代码、FFI、裸指针、未定义行为、内存安全、环境变量、线程安全实现'
---

# Unsafe Rust Reviewer

## Core Question

Can each unsafe operation, unsafe trait implementation, and FFI boundary be backed by a local invariant proof that survives errors, panics, threads, and cleanup? Classify the target before choosing the review depth.

## Core Rules

Treat every `unsafe` block, `unsafe fn`, unsafe trait implementation, and FFI boundary as a proof obligation.

1. **Pointer validity** — Prove provenance, alignment, initialization, lifetime, and bounds for every dereference.
2. **Aliasing** — Preserve Rust's exclusive-mutation and shared-read rules; justify raw-pointer reborrows and `Send`/`Sync` implementations.
3. **Layout and initialization** — Verify `repr(C)`, field order, padding, ownership, drop behavior, `MaybeUninit`, unions, and transmute size/alignment.
4. **FFI boundary** — Match ABI, calling convention, nullability, ownership, string encoding, thread-safety, and unwind behavior on both sides.
5. **Safe surface** — Keep unsafe code small; expose a safe wrapper only when all caller-observable preconditions are enforced.

## Review Process

Classify the target first:

- **Test-only global-state or lifetime manipulation**: run one isolation check—verify the invariant for the test lifetime and prevent side effects from leaking between tests. For environment mutation, require a single-threaded or serialized context and restore the previous value, including the unset case.
- **Production unsafe block, `unsafe fn`, or FFI boundary**: run all five review steps below.
- **`unsafe impl Send/Sync`**: verify thread-safety invariants and absence of interior-mutability races; apply the production path to related unsafe operations.

For production code:

1. State the invariant required before each unsafe operation.
2. Identify who establishes, preserves, and invalidates each invariant.
3. Check error, panic, cancellation, and early-return paths for leaks or invalid cleanup.
4. Check `SAFETY` text against the actual proof; it must explain why the operation is sound, not restate the code.
5. Report definite unsoundness separately from portability or maintainability concerns.

## Low-Unsafe Codebase Checklist

When an initial scan finds no unsafe code or only one isolated unsafe block, narrow the review and record the scope:

1. Confirm there is no `#[no_mangle]`, `extern "C"`, or `cc::Build` in `build.rs`.
2. Check `Cargo.toml` for native or `-sys` dependencies that may hide FFI/native unsafe behind a safe API.
3. Verify `Send`/`Sync` implementations are compiler-derived; flag every manual `unsafe impl` for a dedicated proof.
4. Audit `std::mem::forget`, `ManuallyDrop`, and `MaybeUninit`; their APIs may be safe to call, but ownership, initialization, and cleanup can still be incorrect.

If these checks are clear, report that no direct unsafe/FFI findings were found and state what was covered. Do not manufacture pointer-level findings for code that has no such operation.

## Common Mistakes

| Finding | Correction |
|---|---|
| Dereference justified only by `!is_null()` | Also prove alignment, initialization, provenance, and lifetime |
| `transmute` used for parsing or layout conversion | Use a checked conversion or an explicit `repr(C)`/byte-level operation |
| `extern "C"` wrapper ignores ownership or nullability | Document and enforce both sides of the contract |
| `unsafe impl Send/Sync` has no invariant proof | Prove thread-safety or remove the impl |
| FFI callback can unwind into foreign code | Catch or prohibit unwinding at the boundary |
| Full function marked `unsafe` for one operation | Isolate the smallest unsafe block and validate inputs first |
| Test-only `set_var`/`remove_var` without paired restoration | Verify a single-threaded or serialized context and restore the prior value; leaky environment variables poison subsequent tests. Prefer a scoped environment helper where possible |
| `mem::forget` or `ManuallyDrop` treated as harmless because the call is safe | Audit skipped destructors, ownership transfer, and cleanup on every exit path |

## Boundary

Use `resource-lifecycle` for ordinary `Drop`, lease, and cleanup design; use `concurrency-testing` for deterministic interleavings and test isolation; use `rust-ecosystem` for native dependency and build portability review. Keep this skill focused on the soundness proof at the unsafe or FFI boundary.
