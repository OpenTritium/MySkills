---
name: rust-ecosystem
description: 'Review Rust Cargo and crate integration for compatibility, feature discipline, build portability, and dependency risk. Use when changing Cargo.toml, adding or upgrading crates, selecting features, setting MSRV or edition, using build.rs/proc macros, or resolving integration failures. 中文触发：依赖管理、Cargo、crate、feature、版本升级、供应链'
---

# Rust Ecosystem Reviewer

## Core Rules

1. **Compatibility first** — Check edition, MSRV, target platforms, runtime versions, and public API constraints before selecting a crate or upgrade.
2. **Feature discipline** — Enable only required features; inspect default features and transitive effects. Keep optional integrations optional in libraries.
3. **Dependency hygiene** — Distinguish direct from transitive dependencies; review semver range, lockfile changes, release history, license, and security posture.
4. **Build boundaries** — Audit `build.rs`, proc macros, native libraries, environment assumptions, generated code, and cross-compilation behavior.
5. **Integration contract** — Verify trait versions, error types, runtime requirements, serialization formats, and feature-gated APIs together; do not solve a mismatch by broadening versions blindly.

## Review Process

1. State the required capability and supported targets.
2. Inspect candidate crate versions, features, MSRV, and transitive dependency impact.
3. Check `Cargo.lock`, build scripts, generated/native code, and CI coverage.
4. Report compatibility, maintenance, supply-chain, and migration risks separately.

## Common Mistakes

| Finding | Correction |
|---|---|
| Add a crate for a small standard-library task | Compare maintenance and dependency cost first |
| Enable a feature bundle “just in case” | Enable the smallest feature set and test the selected target |
| Upgrade one crate without checking the lockfile | Review the complete dependency graph and MSRV impact |
| Hide a native requirement in `build.rs` | Document detection, failure, and cross-compilation behavior |
| Use a transitive crate directly | Declare a direct dependency when the code imports it |
| Resolve version conflict with broad ranges | Align compatible versions or isolate the integration boundary |
