---
name: naming-smell
description: Use when reviewing variable/function/type names in code — detecting generic names, misleading names, noise words, abbreviations, single-letter variables, boolean blindness. Keywords: naming, variable name, function name, type name, rename, code smell, readability, data, tmp, info, result, handler, manager, 命名, 变量命名, 可读性
---

# Naming Smell Reviewer

## Overview
Names are the most frequent documentation developers read. A good name is a micro-spec: it answers "what is this?" without looking at its implementation. A bad name lies, hides, or says nothing. Code review must treat naming as a first-class correctness concern.

## When to Use
- Reviewing new variable/function/type names in PRs
- Encountering `data`, `tmp`, `info`, `result`, `val`, `item` in production code
- Functions named `process()`, `handle()`, `run()`, `do_it()`
- Boolean parameters or `bool` returns passed around without context
- Auditing legacy codebase for readability debt

## Rules Engine

1. **Reveal Intent** — The name must answer WHAT the thing IS or DOES, not its type or storage mechanism. `user_ids` beats `list`; `total_revenue` beats `sum`.

2. **No False Advertising** — `customers: Vec<_>` named `customer_set` is a lie. `validate_or_throw` that doesn't is a trap. Names are contracts; discrepancies are bugs. The compiler enforces types; nothing enforces names but review.

3. **Kill Noise Words** — Strip filler: `Data`, `Info`, `Object`, `Thing`, `Item`. Rust's type system already carries `Vec`/`HashMap`/`String` — don't duplicate it in the name (`user_list: Vec<User>` → `users: Vec<User>`).

4. **Verbs for Functions, Nouns for Values** — `get_user()` not `user()`. `is_expired()` not `expired()` for predicates. Functions DO things; values ARE things.

5. **Abbreviation Gate** — Only universally recognized abbreviations survive: `url`, `http`, `sql`, `json`, `db`, `ctx` (convention). Kill `usr`, `pwd`, `msg`, `cnt`, `idx`, `mgr`. Follow the existing codebase's convention where one exists.

6. **No Boolean Blindness** — A bare `bool` at a call site tells you nothing about what `true` means. `set_enabled(device, true)` — true means on? enabled? active? Three fixes, in order of preference:
   - **Two functions**: `enable(device)` / `disable(device)` — the call site reads itself.
   - **Named enum**: `set_mode(device, Mode::Enabled)` — every value is self-documenting.
   - **Boolean only with an interrogative name**: `set(device, is_enabled: bool)` — acceptable when the name mirrors the semantics, but still weaker than the enum.
   This is the same smell as boolean *parameters* (see `func-smell` Rule 2); here it's about the name/type, there about the function design.

## Signal Names

| Category | Rule | Good | Bad |
|---|---|---|---|
| Variable | Noun, specific | `active_users`, `retry_timeout` | `data`, `list`, `tmp` |
| Function | Verb + noun | `send_invoice()`, `parse_config()` | `process()`, `handle()` |
| Boolean | Predicate (`is_`/`has_`/`can_`) | `is_ready`, `has_permission`, `can_retry` | `flag`, `status`, `ready` |
| Boolean param | Avoid; if used, name the meaning | `set(device, is_enabled)` | `set(device, val)` |
| Constant | UPPER_SNAKE with meaning | `MAX_RETRY_ATTEMPTS` | `THREE`, `VAL_42` |
| Type | Noun describing what it models | `OrderValidator`, `CachePolicy` | `Helper`, `Utils`, `Common` |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `data` / `info` / `result` / `tmp` / `val` | Name WHAT data: `parsed_orders`, `config_overrides`, `matched_count` |
| `users: Vec<User>` named `user_list` | `users` — drop the type suffix the compiler already knows |
| `validate()` returning a boolean | Rename to `is_valid()`, or return `Result` and error on invalid |
| `process(s: String, b: bool)` generic + flag | `process(order_id, mode: Mode)` — see `func-smell` for the design fix |
| `DataManager`, `InfoProcessor`, `BaseHandler` | Delete noise words: `OrderStore`, `ConfigParser`, `RateLimiter` |
| `get_user()` with side effects (DB write, cache update) | `get_` implies pure & cheap; rename to `fetch_`/`load_` (see `func-smell` Rule 5 for whether the side effect belongs) |
| `x1`, `x2`, `response1`, `response2` numbering | Name their roles: `current_response`, `previous_response` |
| `fn set(device: &Device, flag: bool)` boolean blindness | `enable(device)` / `disable(device)`, or `set_mode(device, Mode::...)` |
| Abbreviation overuse: `usr_repo`, `mgr` | `user_repository`, `manager` — clarity > keystrokes |

## Workflow
1. Scan for one/two-character names and `data`/`tmp`/`info` — each is a required rename
2. Audit function names: does the name promise more or less than the implementation delivers?
3. Check boolean names: can you read `if (flag)` and know what it guards?
4. Strip noise suffixes (`Data`, `Info`, `Manager`) — does the name lose information? If not, they were never adding any
5. Output renames with before/after pairs and justification
