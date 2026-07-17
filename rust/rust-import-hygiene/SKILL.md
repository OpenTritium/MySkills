---
name: rust-import-hygiene
description: "Review Rust `use` declarations for explicit local paths, selective enum-variant globs, meaningful qualified names, and collision-safe aliases. Use when simplifying imports, resolving ambiguous names, or choosing between `use` and a qualified path. 中文触发：导入优化、use 导入、枚举变体、限定路径、导入别名"
---

# Rust Import Hygiene

Optimize imports for readable name resolution. Preserve behavior, macro resolution, public re-exports, and useful crate or module context.

## Rules

1. **Glob enum variants selectively** — Use `XEnum::*` only when that enum is the module's dominant vocabulary, its variants are used frequently, and no variant, type, function, or constant collides. Otherwise use named variants or `XEnum::Variant`.

2. **Keep meaningful namespaces** — Leave APIs qualified when the path communicates an execution model or prevents a common-name collision: prefer `tokio::spawn` over importing `spawn`, especially alongside `std::thread::spawn`, `async_std`, or `rayon`.

3. **Make local imports explicit** — Avoid bare module imports such as `use crate::state;` and broad relative globs such as `use super::*` or `use crate::state::*`. For `self::`, `super::`, and `crate::` paths, name the terminal item (`use super::parser::Parser`) or alias a module when the alias adds context (`use super::parser as order_parser`).

4. **Choose by use count and context** — Keep a readable qualified path for a one-off or semantically important use. For repeated use, explicitly import the terminal item and group siblings with braces. Never use `::*` just to shorten a long path.

5. **Alias only to clarify** — Use `as` for a real collision or a meaningful module/domain distinction. Avoid arbitrary abbreviations and aliases that erase the source namespace.

6. **Keep scope narrow** — Import only what the module uses. Preserve intentional `pub use` API re-exports; do not change visibility or re-export private implementation details during cleanup.

## Workflow

Inspect use counts, collisions, namespace context, and public API first. Choose qualified paths, explicit terminal imports, enum-variant globs, or aliases in that order of increasing name shortening. Run `rustfmt` and the narrowest relevant check.

```rust
tokio::spawn(run_worker());

use crate::state::{ConnectionState, Event};
use super::parser::Parser;
use ConnectionState::*;

match state {
    Connected => handle_connected(),
    Disconnected => handle_disconnected(),
}
```

If `Connected` or `Disconnected` can collide, replace the glob with `use ConnectionState::{Connected, Disconnected};` or keep the variants qualified.

Report the import, concrete evidence, smallest replacement, and validation. Do not reformat unrelated code or alter public API shape solely for style.
