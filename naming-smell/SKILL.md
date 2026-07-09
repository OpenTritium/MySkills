---
name: naming-smell
description: Use when reviewing variable/function/type names in code — detecting generic names, misleading names, noise words, abbreviations, single-letter variables, boolean blindness. Keywords: naming, variable name, function name, type name, rename, code smell, readability, data, tmp, info, result, handler, manager, 命名, 变量命名, 可读性
---

# Naming Smell Reviewer

## Overview
Names are the most frequent documentation developers read. A good name is a micro-spec: it answers "what is this?" without looking at its implementation. A bad name lies, hides, or says nothing. Code review must treat naming as a first-class correctness concern.

## When to Use
- Reviewing new variable/function/type names in PRs
- Encountering `data`, `tmp`, `info`, `result` in production code
- Functions named `process()`, `handle()`, `run()`, `do()`
- Auditing legacy codebase for readability debt

## Rules Engine

1. **Reveal Intent** — The name must answer WHAT the thing IS or DOES, not its type or storage mechanism. `userIDs` beats `list`; `totalRevenue` beats `sum`.

2. **No False Advertising** — `customerList` when it's a Set is a lie. `validateOrThrow` when it doesn't throw is a trap. Names are contracts; discrepancies are bugs.

3. **Kill Noise Words** — Strip filler: `Data`, `Info`, `Object`, `Thing`, `Item`. If every variable has a type suffix (`Str`, `Int`, `List`), delete them — the type system already enforces this.

4. **Verbs for Functions, Nouns for Values** — `getUser()` not `user()`. `isExpired()` not `expired()`. Functions DO things; booleans ASK questions.

5. **Abbreviation Gate** — Only universally recognized abbreviations survive: `URL`, `HTTP`, `SQL`, `JSON`. Kill `usr`, `pwd`, `msg`, `cnt`, `idx`.

## Signal Names

| Category | Rule | Good | Bad |
|---|---|---|---|
| Variable | Noun, specific | `activeUsers`, `retryTimeout` | `data`, `list`, `tmp` |
| Function | Verb + noun | `sendInvoice()`, `parseConfig()` | `process()`, `handle()` |
| Boolean | Question or predicate | `isReady`, `hasPermission`, `canRetry` | `flag`, `status`, `ready` |
| Constant | UPPER_SNAKE with meaning | `MAX_RETRY_ATTEMPTS` | `THREE`, `VAL_42` |
| Type/Class | Noun describing what it models | `OrderValidator`, `CachePolicy` | `Helper`, `Utils`, `Common` |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `data` / `info` / `result` / `tmp` | Name WHAT data: `parsedOrders`, `configOverrides`, `matchedCount` |
| `list` / `map` / `set` without domain meaning | `activeUsers` (Set), `orderIDToTotal` (Map) |
| `validate()` returning a boolean | Rename to `isValid()` or return error on invalid |
| `process(w string, h bool)` single-letter params | `process(orderID, dryRun)` |
| `DataManager`, `InfoProcessor`, `BaseHandler` | Delete noise words: `OrderStore`, `ConfigParser`, `RateLimiter` |
| `getXxx()` with side effects (DB write, cache update) | Rename to `fetchXxx()` or `loadXxx()` — getter implies pure |
| `x1`, `x2`, `response1`, `response2` numbering | Name their roles: `currentResponse`, `previousResponse` |
| Abbreviation overuse: `usrRepo`, `mgr` | `userRepository`, `manager` — clarity > keystrokes |

## Workflow
1. Scan for one/two-character names and `data`/`tmp`/`info` — each is a required rename
2. Audit function names: does the name promise more or less than the implementation delivers?
3. Check boolean names: can you read `if (flag)` and know what it guards?
4. Strip noise suffixes (`Data`, `Info`, `Manager`) — does the name lose information? If not, they were never adding any
5. Output renames with before/after pairs and justification
