---
name: func-smell
description: Use when reviewing function/method design — parameter explosion, boolean flags, side effects, deep nesting, god functions, output arguments, long functions, mixed abstraction levels. Keywords: function, method, parameter, boolean flag, side effect, nesting, god function, long method, cyclomatic complexity, SRP, single responsibility, refactor, extract method, 函数, 方法, 重构
---

# Function Smell Reviewer

## Overview
A function is a contract: given these inputs, produce this output or effect. When that contract leaks — through flag parameters that split behavior, side effects the name hides, or a body so long you need a map — the function is lying. Review functions for contract integrity, not just correctness.

## When to Use
- Functions with >4 parameters or >50 lines
- Boolean parameters that toggle behavior (`process(orders, dryRun)` )
- Functions doing I/O, validation, and business logic in one body
- Deeply nested code (>3 levels of `if`/`for`/`try`)
- Functions modifying input parameters via pointers/references

## Rules Engine

1. **Parameter Budget** — 3 is great, 4 is suspect, 5+ demands a parameter object or functional split. Each parameter is a dimension of coupling; every flag doubles the states to test.

2. **No Boolean Flags** — `save(order, true)` means nothing at the call site. Split into `saveOrder(order)` and `saveOrderDryRun(order)`, or use a config struct. Boolean flags are the #1 signal that a function is really two functions in a trench coat.

3. **One Level of Abstraction** — A function should not mix `openFile()` with `computeTax()` with `sendEmail()`. Either it orchestrates (calls high-level steps) or it computes (pure logic). Never both in the same function body.

4. **Depth Budget** — Nesting beyond 3 levels (`if` inside `for` inside `if` inside `try`) is unreadable. Extract inner logic to named functions; the name documents the purpose and the structure flattens.

5. **Side Effect Declaration** — If a function writes to a DB, publishes to a queue, or mutates a global, its name MUST reveal it. `getUser()` that updates a cache is a lie; `fetchAndCacheUser()` is honest.

6. **Command vs Query** — A function either changes state (command) OR returns data (query), never both. `pop()` is the classic violator. Split into `peek()` + `remove()` or return a copy.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `createOrder(items, user, tx, dryRun, async, priority)` 6 params | Group into `OrderRequest` struct with meaningful field names |
| `func process(flag bool)` — what does `process(true)` mean? | Split: `func processWithValidation()` and `func processBare()` |
| 300-line function with 8 levels of nesting | Extract blocks into named helper functions until top-level reads like pseudocode |
| Function named `validateAndSave()` doing both | Split into `validate()` returning errors + `save()`; call separately |
| `func getUser(id)` that writes to metrics/updates LRU cache | Rename: `func fetchUser(id)` or document side effects in name |
| Mutable output parameter: `func parse(s string, out *Result)` | Return `(Result, error)` by value instead |
| Early returns mixed with deep else-if chains | Invert conditions: `if invalid { return }` flattens the happy path |
| `doEverything()` function mixing SQL queries, JSON parsing, email sending | Extract per-responsibility: `loadOrders()`, `parseOrderBody()`, `notifyCustomer()` |

## Workflow
1. Identify functions exceeding parameter or length budgets — flag for refactor
2. Replace every boolean flag parameter with a named alternative (split function or config struct)
3. Extract deepest nesting levels into named helper functions
4. Audit function names against side effects: update names to match behavior
5. Split query+command functions into pure query + pure command
6. Output refactored structure with function signatures before/after
