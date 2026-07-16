---
name: testing-strategy
description: 'Review Rust test strategy for behavioral coverage, determinism, failure diagnosis, and appropriate test levels. Use when adding or auditing unit, integration, async, property, snapshot, compile-fail, or fuzz tests, flaky tests, mocks, coverage, or test isolation. 中文触发：测试策略、单元测试、集成测试、属性测试、模糊测试、 flaky 测试'
---

# Rust Testing Reviewer

## Core Rules

1. **Test the contract** — Assert observable behavior, errors, state transitions, and compatibility; avoid coupling tests to private implementation details.
2. **Cover boundaries** — Include invalid input, empty/max values, retries, cancellation, time, I/O failure, and concurrent interleavings when they are part of the contract.
3. **Keep tests deterministic** — Control time, randomness, network, filesystem, environment, and scheduling. Avoid sleeps and shared mutable global state.
4. **Choose the right level** — Unit-test pure logic; integration-test wiring and persistence; use property/fuzz tests for invariants; use compile-fail tests for type/API guarantees.
5. **Make failures diagnostic** — Assert exact error variants or meaningful predicates; include inputs and expected outcomes; do not reduce a test to `is_ok()`.
6. **Test async lifecycle** — Exercise timeout, cancellation, task shutdown, and cleanup rather than only the happy path.

## Review Process

1. State the behavior each test protects.
2. Map boundary cases and failure paths to tests.
3. Identify nondeterminism and hidden external dependencies.
4. Remove redundant tests; add the smallest missing test at the lowest useful level.

## Common Mistakes

| Finding | Correction |
|---|---|
| Test only asserts `is_ok()` | Assert value, variant, side effect, or invariant |
| Sleeps to wait for async work | Use a signal, controlled clock, or bounded synchronization |
| Mock reproduces the implementation | Mock the external contract and test real wiring separately |
| One giant integration test covers everything | Split by contract boundary; keep a small end-to-end path |
| Random or order-dependent test data | Seed randomness and isolate state; prefer deterministic fixtures |
| Snapshot hides unstable fields | Normalize timestamps, IDs, paths, and platform-specific output |
