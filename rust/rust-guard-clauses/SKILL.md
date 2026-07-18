---
name: rust-guard-clauses
description: "Review Rust control-flow shape for fail-fast guards, early returns, `?`, `let-else`, validation order, and clear error paths without flattening meaningful state handling. Use when reducing nested conditionals or choosing between a guard, `if let`, `let-else`, and `match`; use func-smell for broader function-contract problems and error-silence for lost error meaning. 中文触发：guard、守卫语句、提前返回、尽早失败、let else、错误路径、减少嵌套"
---

# Rust Guard Clauses

Make invalid or unavailable inputs leave early so the main path stays shallow. Preserve error meaning, ownership, cleanup, and the distinction between expected absence and broken invariants.

## Rules

1. **Reject early** — Check cheap, independent preconditions before expensive work, I/O, mutation, or resource acquisition. Keep dependency-driven validation order and side effects intact.

2. **Propagate deliberately** — Use `?` when the caller owns the error policy. Add context before propagation when needed; do not replace a useful domain error with a generic guard error.

3. **Use `let-else` for required patterns** — Choose `let Some(value) = expr else { return ... };` when failure must diverge from the current function, loop, or scope and the binding drives the main path. Use `break` or `continue` for loop guards when that is the actual outcome.

4. **Match the construct to the branches** — Use `if` for boolean rejection, `if let` when the optional branch is a side action and both paths continue, and `match` when multiple variants have meaningful behavior or exhaustiveness matters.

5. **Keep one cohesive success path** — Flatten accidental nesting, but do not create many tiny returns that fragment the operation or hide its business sequence. Extract a named predicate when a guard condition is long or domain-heavy.

6. **Keep guards observable** — Avoid expensive work, hidden mutation, logging, or lock acquisition inside conditions. Validate before acquiring locks or transactions; do not hold them across unrelated work or `.await`.

7. **Reserve panics for invariants** — Never use `unwrap` or `expect` as validation for user input, external data, or expected absence. Return a typed error, `None`, or an explicit fallback instead.

## Example

```rust
fn parse_name(input: &str) -> Result<Name, ParseError> {
    let Some(line) = input.lines().find(|line| !line.trim().is_empty()) else {
        return Err(ParseError::MissingName);
    };

    let name = Name::parse(line)?;
    validate_name(&name)?;
    Ok(name)
}
```

Use `match` instead when each state has a meaningful outcome:

```rust
match connection.state() {
    ConnectionState::Ready => send_request(connection),
    ConnectionState::Closed => Err(Error::Closed),
    ConnectionState::Backoff(until) => retry_after(until),
}
```

Report the delayed failure or nesting, smallest safe refactor, chosen construct, and preserved error, ownership, async, or cleanup boundary. Do not add early returns merely to reduce line count.
