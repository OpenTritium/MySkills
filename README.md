# MySkills

Codex skills for Rust development, code review, and safe version-control workflows.

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

## Organization

Skills remain under stable source roots, but the index groups them by the question they answer. Load one primary skill first and add a secondary skill only when the request crosses its documented boundary.

### Language And Runtime

- `async-concurrency` — async tasks, synchronization, cancellation, and backpressure
- `encode-invariant` — type-level immutability and domain invariants
- `error-silence` — Rust error propagation and panic boundaries
- `resource-lifecycle` — ownership, cleanup, pools, transactions, and shutdown
- `rust-state-machine` — explicit Rust states, legal transitions, and invalid-state prevention
- `unsafe-checker` — unsafe Rust and FFI soundness

### Design And Architecture

- `architecture-entropy-review` — architectural drift after large refactors
- `rust-api-consolidation` — merge or remove Rust APIs while preserving real seams
- `rust-ecosystem` — Cargo, crate integration, features, compatibility, and dependency risk
- `rust-structure-refactor` — function, struct, and module-boundary refactoring

### Review And Clarity

- `big-o-optimizer` — algorithm complexity and data structures
- `func-smell` — function and method design
- `high-snr-comment` — comments and documentation signal quality
- `high-snr-log` — structured log content and noise
- `log-level` — tracing levels and OpenTelemetry semantics
- `naming-smell` — variable, function, and type naming
- `rust-guard-clauses` — fail-fast guards, `let-else`, `?`, and shallow control flow
- `rust-import-hygiene` — Rust `use` imports, qualified paths, enum globs, and aliases
- `testing-strategy` — behavioral, deterministic, async, property, and integration tests
- `zero-alloc` — avoidable allocations in Rust hot paths

### Version Control

- `jujutsu` — safe agent workflows for Jujutsu version control
- `jujutsu-parallel` — parallel Jujutsu workspaces for multiple agents

See [test-triggers.md](test-triggers.md) for primary skill ownership and overlap boundaries. See [AGENTS.md](AGENTS.md) for contribution and validation rules.

## Adding Or Updating A Skill

1. Give the skill one primary contract and a trigger description.
2. Add it to the appropriate organization category.
3. Add representative trigger cases and document any secondary boundary.
4. Keep the body concise and link conditional detail instead of duplicating it.
5. Run the skill validator before handoff.
