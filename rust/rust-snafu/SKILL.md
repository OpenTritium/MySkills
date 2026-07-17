---
name: rust-snafu
description: "Design and review Rust error types with Snafu, including `#[derive(Snafu)]`, context selectors, `context`, `ensure!`, `Whatever`, sources, backtraces, opaque public errors, and feature flags. Use when adding or reviewing Snafu-based error handling, context propagation, error reporting, or no_std/library API boundaries. 中文触发：Snafu、snafu、错误上下文、context selector、Whatever、ensure、错误类型、错误链、backtrace"
---

# Rust Snafu

Keep domain meaning and source context as errors cross module boundaries. Use Snafu to make that contract explicit, not merely to shorten `map_err` calls.

## Choose The Error Shape

| Situation | Prefer |
|---|---|
| Application prototype or heterogeneous, stringly errors | `Whatever`; replace it when callers need typed matching |
| One constrained failure with one context shape | A `struct` deriving `Snafu` |
| Several domain failures in one module | An `enum` deriving `Snafu` |
| Stable public API whose variants should remain private | A public opaque newtype around a private Snafu error |
| Wrapper adds no context and should expose the source unchanged | `#[snafu(transparent)]` |

Keep error types close to the module that owns the operation. Do not expose internal variants merely so callers can construct context selectors.

## Core Rules

1. **Describe every public failure** — Add `#[snafu(display(...))]` with actionable domain context. Do not include secrets, unstable internals, or an error message that omits the operation's identity.
2. **Preserve the source chain** — Use a `source` field for the underlying error and propagate with the generated selector: `.context(ReadConfigSnafu { path })?`. Use `#[snafu(source)]` only when the field has another name; use `source(from(...))` only for an intentional conversion.
3. **Use selectors for local failures** — Use `ensure!(predicate, Selector { fields })` for expected validation failures and `Selector { fields }.fail()`/`.build()` when a direct error is clearer. Use `with_context` only when constructing context is expensive or conditional.
4. **Report at the boundary** — `Display` shows the top-level message; use `snafu::Report` or `#[snafu::report]` at the application boundary to render the source chain and backtrace. Add context in intermediate layers and avoid logging the same error repeatedly.
5. **Keep generated APIs narrow** — Context selectors are private by default. Use `#[snafu(visibility(pub(crate)))]` only for a real module boundary, `#[snafu(module)]` for deliberate namespacing, and `context(false)` when a unique source needs no additional context.
6. **Capture backtraces deliberately** — Add a `Backtrace` field where diagnostics justify the cost; use `#[snafu(backtrace)]` when delegating to a source's backtrace. Do not add backtraces to every variant automatically.
7. **Control features by target** — Keep default `std`/`alloc` behavior unless the crate needs `no_std`. Enable `futures` only for future/stream extensions; treat provider, unstable try-trait, and alternate backtrace integrations as deliberate application-level choices.

```rust
use snafu::prelude::*;

#[derive(Debug, Snafu)]
enum Error {
    #[snafu(display("Could not read config at {path}"))]
    ReadConfig { source: std::io::Error, path: String },
    #[snafu(display("port {port} is invalid"))]
    InvalidPort { port: u16 },
}

fn load(path: &str, port: u16) -> Result<String, Error> {
    ensure!(port != 0, InvalidPortSnafu { port });
    std::fs::read_to_string(path)
        .context(ReadConfigSnafu { path: path.to_owned() })
}
```

## Validation

Check the error variant, `Display` output, source chain, boundary report, visibility, feature configuration, and public API stability. Test behavior and error contracts, not the exact generated selector implementation. Use `cargo expand` only to investigate macro output.

Use `error-silence` for swallowed errors, panic sites, and duplicate logging; use `rust-ecosystem` for Snafu version, feature, MSRV, and dependency review.

Official references: [Snafu guide](https://docs.rs/snafu/latest/snafu/guide/), [`Snafu` derive](https://docs.rs/snafu/latest/snafu/derive.Snafu.html), [`Whatever`](https://docs.rs/snafu/latest/snafu/struct.Whatever.html), and [the official repository](https://github.com/shepmaster/snafu).
