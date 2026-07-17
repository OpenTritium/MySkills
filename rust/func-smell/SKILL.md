---
name: func-smell
description: 'Review function and method design for parameter explosion, boolean flags, hidden side effects, deep nesting, god functions, output arguments, long bodies, and mixed abstraction levels. Keywords: function, method, parameter, boolean flag, side effect, nesting, god function, long method, cyclomatic complexity, SRP, single responsibility, refactor, extract method, 函数, 方法, 重构'
---

# Function Smell Reviewer

## Overview
A function is a contract: given inputs, produce this output or effect. When the contract leaks — flag params splitting behavior, side effects the name hides, a body so long you need a map — the function is lying. Review for contract integrity, not just correctness.

## Rules Engine

1. **Parameter Budget** — Treat 4+ parameters as a review trigger, not an automatic defect. Group parameters only when they share a concept or lifecycle; split behavior when the contract is genuinely different.

2. **Review Boolean Flags at the Call Site** — A `bool` is a review trigger, not an automatic defect. Flag `save(order, true)` when the argument is ambiguous or selects materially different contracts. Prefer named operations, a config struct, or `enum Mode` in that case. Keep a boolean when its source communicates the predicate clearly, such as `record(metrics, result.is_ok())`, and splitting it would only duplicate implementation. Do not force two APIs merely because a parameter has type `bool`.

3. **One Level of Abstraction** — Don't mix `open_file()` with `compute_tax()` with `send_email()`. Either orchestrate (high-level steps) or compute (pure logic), not both.

4. **Depth Budget** — More than 3 levels of `if`/`for`/`match` often deserves inspection. Extract when a named helper clarifies a unit of work; guard clauses (`if invalid { return }`) can flatten the happy path.

5. **Side Effect Declaration** — Make important I/O and mutation discoverable in the API or name. `get_user()` that updates a cache may surprise callers; consider `fetch_and_cache_user()` or a documented convention. (`naming-smell` owns the rename; this rule owns whether the side effect belongs.)

6. **Command vs Query** — Inspect functions that both mutate and return data. Combining them can be valid when the returned value is the mutation's result (`Vec::pop()`); split only when the responsibilities or callers are independent.

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `create_order(items, user, tx, dry_run, async_mode, priority)` | Group into `OrderRequest` struct |
| `fn process(flag: bool)` called as `process(true)` — what's `true`? | Split `process()` / `process_dry_run()`, or use `enum Mode`; retain a predicate argument when call sites already explain it |
| 300-line fn, 8 levels nesting | Extract helpers until top level reads like pseudocode |
| `validate_and_save()` doing both | Split `validate() -> Result<()>` + `save()` |
| `fn get_user(id)` writing metrics / updating LRU | Split the side effect out, or rename `fetch_and_cache_user()` (`naming-smell`) |
| `fn parse(s: &str, out: &mut Result)` | Return `Result<Parsed, Error>` by value |
| `fn process(&mut self) -> Stats` mixes unrelated mutation and reporting | Split when callers need either operation independently |
| `do_everything()` mixing SQL, JSON, email | Extract: `load_orders()`, `parse_body()`, `notify_customer()` |

## Workflow
1. Flag functions exceeding parameter or length budgets
2. Review boolean parameters at their call sites; split only when the value is ambiguous or the contracts genuinely diverge
3. Extract deepest nesting into named helpers
4. Audit names against side effects — update to match behavior
5. Split query+command functions
6. Output signatures before/after
