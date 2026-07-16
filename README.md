# MySkills

Rust-focused Codex skills for code review, correctness, observability, naming, and performance.

## Install with Bun

This repository keeps skills under `rust/`, so pass `--full-depth` to the `skills` CLI.

List the available skills:

```bash
bunx skills add OpenTritium/MySkills --list --full-depth
```

Install every skill into the current project's Codex directory (`.agents/skills`):

```bash
bunx skills add OpenTritium/MySkills \
  --agent codex \
  --skill '*' \
  --full-depth \
  --copy \
  --yes
```

Install one skill:

```bash
bunx skills add OpenTritium/MySkills \
  --agent codex \
  --skill log-level \
  --full-depth \
  --copy \
  --yes
```

Verify the project installation:

```bash
bunx skills list --agent codex
```

Refresh the shared global `$HOME/.agents/skills` directory:

```bash
rm -rf "$HOME/.agents/skills"/*
bunx skills add OpenTritium/MySkills \
  --global \
  --agent cline \
  --skill '*' \
  --full-depth \
  --copy \
  --yes
```

The `cline` target is used here because this CLI agent maps its global skills directory to `$HOME/.agents/skills`.

To refresh a dedicated project `.agents/skills` directory, remove its contents and run the install command again. Do not remove a shared directory that contains skills from other repositories.

## Skills

- `big-o-optimizer` — algorithm complexity and data structures
- `async-concurrency` — async tasks, synchronization, cancellation, and backpressure
- `encode-invariant` — type-level immutability and domain invariants
- `error-silence` — Rust error propagation and panic boundaries
- `func-smell` — function and method design
- `high-snr-comment` — comments and documentation signal quality
- `high-snr-log` — structured log content and noise
- `log-level` — tracing levels and OpenTelemetry semantics
- `naming-smell` — variable, function, and type naming
- `resource-lifecycle` — ownership, cleanup, pools, transactions, and shutdown
- `rust-structure-refactor` — function, struct, and module-boundary refactoring
- `rust-ecosystem` — Cargo, crate integration, features, compatibility, and dependency risk
- `testing-strategy` — behavioral, deterministic, async, property, and integration tests
- `unsafe-checker` — unsafe Rust and FFI soundness
- `zero-alloc` — avoidable allocations in Rust hot paths
