---
name: writing-tunit-tests
description: "Write or review TUnit tests for .NET 10, including async assertions, parameterized cases, categories, lifecycle, skips, and isolated fixtures. Do not use for running tests, broad test audits, or unrelated test tooling."
license: MIT
---

# Writing TUnit Tests

Use this skill for concrete TUnit source changes in a .NET 10 project. TUnit
tests are convention based and asynchronous by design. The most dangerous
TUnit mistake is an assertion that is not awaited: the test can finish without
executing the assertion.

## Core question

Does the test express one observable behavior, await every assertion and async
operation, and isolate its resources from other tests?

## When to use

- Write or fix tests using the `TUnit` package.
- Replace weak assertions or incorrect exception checks.
- Add `[Arguments]`, data sources, categories, skip reasons, or lifecycle
  hooks.
- Review TUnit-specific async, fixture, and parallelism choices.

## Boundary

- Use `run-tests` to construct or execute the TUnit command.
- Use a test-audit skill for broad coverage, smell, mutation, or flakiness
  analysis; this skill changes concrete TUnit tests only.
- Keep this skill focused on TUnit APIs and the .NET 10 test project.

## Minimal structure

TUnit discovers `[Test]` methods without a test-class marker. Keep test methods
async when the code under test is async and use descriptive names:

```csharp
using TUnit.Core;

public class OrderTests
{
    [Test]
    public async Task CalculateTotal_WithDiscount_ReturnsReducedPrice()
    {
        var service = new OrderService();

        var actual = await service.CalculateTotalAsync(100m, 10m);

        await Assert.That(actual).IsEqualTo(90m);
    }
}
```

Test classes do not need to be sealed or decorated with `[TestClass]`. Keep
construction cheap and deterministic; put owned resources behind explicit
setup/teardown rather than shared mutable statics.

## Assertions

All TUnit assertions return an awaitable assertion result. Always await them:

| Intent | TUnit form |
|---|---|
| Equality | `await Assert.That(actual).IsEqualTo(expected)` |
| Boolean | `await Assert.That(value).IsTrue()` / `.IsFalse()` |
| Nullability | `await Assert.That(value).IsNull()` / `.IsNotNull()` |
| Collection membership | `await Assert.That(items).Contains(item)` |
| String content | `await Assert.That(text).Contains("part")` |
| Type | `await Assert.That(value).IsAssignableTo<T>()` |
| Exception | `await Assert.That(() => action()).Throws<T>()` |

Prefer the most specific assertion. Do not use a generic boolean assertion for
collection shape, nullability, or string content when a typed assertion makes
the failure clearer. For exception tests, assert the exception type first and
then stable properties such as a parameter name or error code.

For async actions, pass a delegate that returns the task and await the TUnit
exception assertion. Do not wrap an async operation in `async void`, and do not
forget to await a returned assertion merely because the test method itself is
async.

## Data and test metadata

- Use `[Arguments(...)]` for small, readable input variations and a method data
  source for larger or generated cases. Give cases meaningful values or names
  so a failure identifies its input.
- Use `[Category("Unit")]`, `[Category("Integration")]`, or another project
  taxonomy consistently. `run-tests` maps these to TUnit `--treenode-filter`
  expressions.
- Use `[Skip("reason")]` only for a known, documented temporary or
  environment-specific exclusion. Never skip a failing test to make a run
  green.

## Lifecycle and isolation

- Use a constructor for cheap per-test setup when it keeps dependencies
  readonly.
- Use `[Before(Test)]` and `[After(Test)]` for TUnit lifecycle work that must
  be explicit; use `[Before(Class)]` and `[After(Class)]` only for genuinely
  shared, safely disposable resources.
- Implement `IDisposable` or `IAsyncDisposable` for owned resources and ensure
  cleanup also runs when setup or the test fails.
- Treat tests as independently runnable. Isolate files, ports, databases,
  environment variables, clocks, and static state before enabling parallel
  execution.
- Avoid arbitrary sleeps. Coordinate async work with a bounded signal,
  timeout, or controllable clock.

## Completion checklist

- [ ] TUnit `[Test]` discovery, awaited assertions, and async production work
      are used correctly.
- [ ] Test data, categories, and fixtures make failures diagnostic and tests
      independently runnable.
- [ ] Resources are isolated and disposed deterministically.
- [ ] The narrow TUnit test command passes.
