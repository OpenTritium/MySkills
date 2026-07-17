---
name: naming-smell
description: "Review Rust variable, function, and type names for misleading contracts, fallibility, optionality, side effects, ownership, blocking behavior, noise, abbreviations, and boolean blindness. Use when naming or reviewing Rust APIs, variables, functions, types, conversions, state methods, or error-returning operations. 中文触发：命名、变量命名、函数命名、API 命名、try 命名、可失败方法、可读性"
---

# Naming Smell Reviewer

Treat names as API contracts. A name should reveal the operation, result, effects, ownership, cost, and failure behavior that callers must understand.

## Rules

1. **Reveal domain intent** — Use specific nouns for values and verbs for operations: `active_users`, `parse_config`, `send_invoice`. Remove storage words (`list`, `data`, `result`) and generic verbs (`process`, `handle`, `do_`) unless context makes them precise.

2. **Name failure and absence honestly** — Do not prefix every `Result` method with `try_`. Use `try_` for a meaningful alternate contract such as `lock`/`try_lock`, `send`/`try_send`, or `new`/`try_new`. Use `find_` when ordinary absence returns `Option`; return `Result` when callers need a failure reason.

3. **Expose effects and cost** — Reserve `get_` for cheap lookup or access; use `fetch_`/`load_` for I/O and `refresh_`/`reconcile_` for synchronization or mutation. Use `ensure_` when the method establishes a condition and `assert_` when it intentionally panics. `is_`/`has_`/`can_`/`should_` should be query-like and side-effect free.

4. **Follow ownership conventions** — Use `as_` for a borrowed view, `to_` for a new value, and `into_` for a consuming conversion. Use `_ref`/`_mut` when borrow mode matters; use `peek`, `take`, `remove`, and `replace` to expose consumption and replacement behavior.

5. **Make paired APIs symmetric** — Name lifecycle operations consistently (`start`/`stop`, `open`/`close`, `enable`/`disable`). Use `blocking_` or `try_` when blocking and immediate/non-blocking variants coexist; follow the codebase's async convention.

6. **Avoid boolean blindness** — Replace ambiguous flags with paired functions, a named enum, or a predicate-named parameter such as `is_enabled`. Do not name booleans `flag`, `status`, or `value`.

7. **Remove false promises and noise** — A name implying purity, cheapness, idempotence, no blocking, or no panic must match the implementation. Drop `Data`, `Info`, `Object`, `Thing`, `Item`, type suffixes, and harmful abbreviations; retain universal abbreviations such as `url`, `http`, `sql`, `json`, `db`, and `ctx` when the codebase agrees.

## Contract Pairs

| Contract | Prefer | Avoid |
|---|---|---|
| Cheap lookup / I/O | `get_user` / `fetch_user` | `get_user` that performs network I/O |
| Predicate / validation / establishment | `is_valid` / `validate` / `ensure_ready` | `check` with an unclear result or effect |
| Borrow / create / consume | `as_path` / `to_path_buf` / `into_path_buf` | `convert` or `make` |
| Non-consuming / move out / delete | `peek` / `take` / `remove` | `read` when it consumes state |
| Fallible alternate | `parse` / `try_parse` only when contracts differ | `try_parse` as decoration |

## Workflow

1. Infer return, absence, mutation, I/O, blocking, ownership, and panic behavior from implementation and call sites.
2. Compare the name with that contract and with neighboring API pairs.
3. Rename only when the new name removes ambiguity; preserve established domain terminology.
4. Check callers, documentation, re-exports, and tests after each rename.
5. Report the before/after name and the contract mismatch that justified the change.
