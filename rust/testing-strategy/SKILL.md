---
name: testing-strategy
description: 'Review Rust test strategy for behavioral coverage, determinism, failure diagnosis, and appropriate test levels. Use when designing or auditing unit, integration, property, snapshot, compile-fail, or fuzz tests, flaky tests, mocks, isolation, behavior characterization, or regression tests. Use async-concurrency for production async/concurrency design and concurrency-testing for forced interleavings. 中文触发：测试策略、行为确认、特征测试、回归测试、happy path、unhappy path、重构前验证、单元测试、集成测试、属性测试、模糊测试、flaky 测试'
---

# Rust Testing Reviewer

## Core Rules

1. **Test the contract** — Assert observable behavior, errors, state transitions, and compatibility; avoid coupling tests to private implementation details.
2. **Cover boundaries** — Include invalid input, empty/max values, retries, cancellation, time, I/O failure, and concurrent interleavings when they are part of the contract.
3. **Keep tests deterministic** — Control time, randomness, network, filesystem, environment, and scheduling. Avoid sleeps and shared mutable global state.
4. **Choose the right level** — Unit-test pure logic; integration-test wiring and persistence; use property/fuzz tests for invariants; use compile-fail tests for type/API guarantees.
5. **Make failures diagnostic** — Assert exact error variants or meaningful predicates; include inputs and expected outcomes; do not reduce a test to `is_ok()`.
6. **Test async lifecycle** — Exercise timeout, cancellation, task shutdown, and cleanup rather than only the happy path.

## Confirm Behavior Before Changing Structure

When a refactor or bug report leaves behavior uncertain, write focused tests through the stable boundary before changing production structure:

1. Cover one relevant happy path and one relevant unhappy path. Add boundary, empty, or cancellation cases only when the contract requires them.
2. For ambiguous existing behavior, write characterization tests that record what the system does, then confirm whether that behavior is intended.
3. For a confirmed bug, write a regression test for the desired behavior; it should fail before the fix and pass after it.
4. Keep the tests after the refactor. They are the contract guard, not temporary probes.

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
