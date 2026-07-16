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

To refresh a dedicated project `.agents/skills` directory, remove its contents and run the install command again. Do not remove a shared directory that contains skills from other repositories.

## Skills

- `big-o-optimizer` — algorithm complexity and data structures
- `encode-invariant` — type-level immutability and domain invariants
- `error-silence` — Rust error propagation and panic boundaries
- `func-smell` — function and method design
- `high-snr-comment` — comments and documentation signal quality
- `high-snr-log` — structured log content and noise
- `log-level` — tracing levels and OpenTelemetry semantics
- `naming-smell` — variable, function, and type naming
- `zero-alloc` — avoidable allocations in Rust hot paths
