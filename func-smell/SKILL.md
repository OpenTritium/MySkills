---
name: func-smell
description: Use when reviewing function/method design — parameter explosion, boolean flags, side effects, deep nesting, god functions, output arguments, long functions, mixed abstraction levels. Keywords: function, method, parameter, boolean flag, side effect, nesting, god function, long method, cyclomatic complexity, SRP, single responsibility, refactor, extract method, 函数, 方法, 重构
---

# Function Smell Reviewer

## Overview
A function is a contract: given these inputs, produce this output or effect. When that contract leaks — through flag parameters that split behavior, side effects the name hides, or a body so long you need a map — the function is lying. Review functions for contract integrity, not just correctness.

## When to Use
- Functions with >4 parameters or >50 lines
- Boolean parameters that toggle behavior (`process(orders, dry_run)`)
- Functions doing I/O, validation, and business logic in one body
- Deeply nested code (>3 levels of `if`/`for`/`match`)
- Functions taking `&mut` output parameters instead of returning values

## Rules Engine

1. **Parameter Budget** — 3 is great, 4 is suspect, 5+ demands a config struct or a functional split. Each parameter is a dimension of coupling; every flag doubles the states to test.

2. **No Boolean Flags** — `save(order, true)` means nothing at the call site. Split into `save()` and `save_dry_run()`, take a config struct, or use an enum (`enum Mode { Real, DryRun }`). Boolean flags are the #1 signal that a function is really two functions in a trench coat.

3. **One Level of Abstraction** — A function should not mix `open_file()` with `compute_tax()` with `send_email()`. Either it orchestrates (calls high-level steps) or it computes (pure logic). Never both in the same function body.

4. **Depth Budget** — Nesting beyond 3 levels (`if` inside `for` inside `match` inside `if`) is unreadable. Extract inner logic to named functions; the name documents the purpose and the structure flattens. Guard clauses (`if invalid { return }`) flatten the happy path.

5. **Side Effect Declaration** — If a function writes to a DB, sends on a channel, or mutates a global, its name MUST reveal it. `get_user()` that updates a cache is a lie; `fetch_and_cache_user()` is honest. (The `naming-smell` skill owns the naming fix — `get` vs `fetch` — this rule owns the design question: should the function have the side effect at all?)

6. **Command vs Query** — A function either changes state (command) OR returns data (query), never both. `Vec::pop()` is the classic violator. Split into `last()` (query) + `truncate()` (command), or return a copy. A `fn process(&mut self) -> Stats` that both mutates and reports is two responsibilities.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `create_order(items, user, tx, dry_run, async_mode, priority)` 6 params | Group into `OrderRequest` struct with named fields |
| `fn process(flag: bool)` — what does `process(true)` mean? | Split into `fn process()` and `fn process_dry_run()`, or take `enum Mode` |
| 300-line function with 8 levels of `match`/`if` nesting | Extract blocks into named helpers until the top level reads like pseudocode |
| Function named `validate_and_save()` doing both | Split into `validate() -> Result<()>` and `save()`; call separately |
| `fn get_user(id)` that writes to metrics / updates an LRU cache | Split the side effect out, or rename to `fetch_and_cache_user()` (see `naming-smell`) |
| Mutable output parameter: `fn parse(s: &str, out: &mut Result)` | Return `Result<Parsed, Error>` by value instead |
| `fn process(&mut self) -> Stats` mutating and reporting | Split: `fn apply(&mut self)` (command) + `fn stats(&self) -> Stats` (query) |
| `do_everything()` mixing SQL, JSON parsing, and email | Extract per-responsibility: `load_orders()`, `parse_body()`, `notify_customer()` |

## Workflow
1. Identify functions exceeding parameter or length budgets — flag for refactor
2. Replace every boolean flag parameter with a named alternative (split function or config struct)
3. Extract deepest nesting levels into named helper functions
4. Audit function names against side effects: update names to match behavior
5. Split query+command functions into pure query + pure command
6. Output refactored structure with function signatures before/after
