---
name: naming-smell
description: Use when reviewing variable/function/type names in code — detecting generic names, misleading names, noise words, abbreviations, single-letter variables, boolean blindness. Keywords: naming, variable name, function name, type name, rename, code smell, readability, data, tmp, info, result, handler, manager, 命名, 变量命名, 可读性
---

# Naming Smell Reviewer

## Overview
Names are the most-read documentation. A good name is a micro-spec — answers "what is this?" without the implementation. A bad name lies, hides, or says nothing. Treat naming as a correctness concern.

## When to Use
- New variable/function/type names in PRs
- `data`, `tmp`, `info`, `result`, `val`, `item` in production code
- Functions named `process()`, `handle()`, `run()`, `do_it()`
- `bool` params/returns without context
- Readability debt audit

## Rules Engine

1. **Reveal Intent** — Answer WHAT it IS/DOES, not its type or storage. `user_ids` beats `list`; `total_revenue` beats `sum`.

2. **No False Advertising** — `customers: Vec<_>` named `customer_set` is a lie. `validate_or_throw` that doesn't is a trap. Names are contracts; the compiler enforces types, review enforces names.

3. **Kill Noise Words** — Strip `Data`, `Info`, `Object`, `Thing`, `Item`. Rust's type already carries `Vec`/`HashMap`/`String` — don't duplicate (`user_list: Vec<User>` → `users`).

4. **Verbs for Functions, Nouns for Values** — `get_user()` not `user()`. `is_expired()` for predicates. Functions DO; values ARE.

5. **Abbreviation Gate** — Keep only universal: `url`, `http`, `sql`, `json`, `db`, `ctx`. Kill `usr`, `pwd`, `msg`, `cnt`, `idx`, `mgr`. Follow existing codebase convention.

6. **No Boolean Blindness** — Bare `bool` says nothing about what `true` means. `set_enabled(device, true)` — on? enabled? active? Fixes, by preference:
   - **Two functions**: `enable(device)` / `disable(device)` — reads itself.
   - **Named enum**: `set_mode(device, Mode::Enabled)` — self-documenting.
   - **Interrogative name**: `set(device, is_enabled)` — acceptable, weaker than enum.
   Same smell as boolean *parameters* (`func-smell` Rule 2); here about name/type, there about design.

## Signal Names

| Category | Rule | Good | Bad |
|---|---|---|---|
| Variable | Noun, specific | `active_users`, `retry_timeout` | `data`, `list`, `tmp` |
| Function | Verb + noun | `send_invoice()`, `parse_config()` | `process()`, `handle()` |
| Boolean | Predicate `is_`/`has_`/`can_` | `is_ready`, `has_permission` | `flag`, `status` |
| Boolean param | Avoid; name meaning if used | `set(device, is_enabled)` | `set(device, val)` |
| Constant | UPPER_SNAKE, meaningful | `MAX_RETRY_ATTEMPTS` | `THREE`, `VAL_42` |
| Type | Noun, models a thing | `OrderValidator`, `CachePolicy` | `Helper`, `Utils`, `Common` |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `data` / `info` / `result` / `tmp` / `val` | Name the data: `parsed_orders`, `config_overrides`, `matched_count` |
| `user_list: Vec<User>` | `users` — drop the type suffix |
| `validate()` returning bool | `is_valid()`, or return `Result` |
| `process(s: String, b: bool)` | `process(order_id, mode: Mode)` (`func-smell`) |
| `DataManager`, `InfoProcessor`, `BaseHandler` | Drop noise: `OrderStore`, `ConfigParser`, `RateLimiter` |
| `get_user()` with side effects | `get_` implies pure/cheap → `fetch_`/`load_` (`func-smell` Rule 5) |
| `x1`, `x2`, `response1`, `response2` | Name roles: `current_response`, `previous_response` |
| `fn set(device, flag: bool)` | `enable(device)` / `disable(device)`, or `set_mode(.., Mode::..)` |
| `usr_repo`, `mgr` | `user_repository`, `manager` |

## Workflow
1. Scan for 1–2 char names and `data`/`tmp`/`info` — each needs a rename
2. Audit function names: do they promise more/less than the impl delivers?
3. Boolean names: does `if (flag)` tell you what it guards?
4. Strip noise suffixes — if the name loses no info, they added none
5. Output before/after pairs with justification
