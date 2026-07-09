---
name: func-smell
description: Use when reviewing function/method design — parameter explosion, boolean flags, side effects, deep nesting, god functions, output arguments, long functions, mixed abstraction levels. Keywords: function, method, parameter, boolean flag, side effect, nesting, god function, long method, cyclomatic complexity, SRP, single responsibility, refactor, extract method, 函数, 方法, 重构
---

# Function Smell Reviewer

## Overview
A function is a contract: given inputs, produce this output or effect. When the contract leaks — flag params splitting behavior, side effects the name hides, a body so long you need a map — the function is lying. Review for contract integrity, not just correctness.

## When to Use
- >4 parameters or >50 lines
- Boolean params toggling behavior (`process(orders, dry_run)`)
- I/O + validation + business logic in one body
- >3 levels of `if`/`for`/`match` nesting
- `&mut` output params instead of return values

## Rules Engine

1. **Parameter Budget** — 3 is great, 4 is suspect, 5+ demands a config struct or functional split. Each param is a coupling dimension; each flag doubles test states.

2. **No Boolean Flags** — `save(order, true)` means nothing at the call site. Split into `save()` / `save_dry_run()`, take a config struct, or use `enum Mode`. Boolean flags signal two functions in a trench coat.

3. **One Level of Abstraction** — Don't mix `open_file()` with `compute_tax()` with `send_email()`. Either orchestrate (high-level steps) or compute (pure logic), not both.

4. **Depth Budget** — >3 levels nesting (`if` in `for` in `match`) is unreadable. Extract to named functions — the name documents purpose, the structure flattens. Guard clauses (`if invalid { return }`) flatten the happy path.

5. **Side Effect Declaration** — If a function writes to DB, sends on a channel, or mutates a global, its name MUST say so. `get_user()` that updates a cache is a lie; `fetch_and_cache_user()` is honest. (`naming-smell` owns the rename; this rule owns: should the side effect be there at all?)

6. **Command vs Query** — Either change state (command) OR return data (query), never both. `Vec::pop()` is the classic violator — split into `last()` + `truncate()`. `fn process(&mut self) -> Stats` is two responsibilities.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `create_order(items, user, tx, dry_run, async_mode, priority)` | Group into `OrderRequest` struct |
| `fn process(flag: bool)` — what's `process(true)`? | Split `process()` / `process_dry_run()`, or `enum Mode` |
| 300-line fn, 8 levels nesting | Extract helpers until top level reads like pseudocode |
| `validate_and_save()` doing both | Split `validate() -> Result<()>` + `save()` |
| `fn get_user(id)` writing metrics / updating LRU | Split the side effect out, or rename `fetch_and_cache_user()` (`naming-smell`) |
| `fn parse(s: &str, out: &mut Result)` | Return `Result<Parsed, Error>` by value |
| `fn process(&mut self) -> Stats` mutating + reporting | Split: `apply(&mut self)` + `stats(&self) -> Stats` |
| `do_everything()` mixing SQL, JSON, email | Extract: `load_orders()`, `parse_body()`, `notify_customer()` |

## Workflow
1. Flag functions exceeding parameter or length budgets
2. Replace boolean flags with named alternatives (split or config struct)
3. Extract deepest nesting into named helpers
4. Audit names against side effects — update to match behavior
5. Split query+command functions
6. Output signatures before/after
