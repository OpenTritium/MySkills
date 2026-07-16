---
name: high-snr-comment
description: 'Review or write Rust code comments and documentation: remove redundant or code-translating comments, stale notes, commented-out code, and empty cheerleading while preserving rationale, contracts, and edge cases. Keywords: comment, annotation, doc, Javadoc, godoc, docstring, ///, //, /*, TODO, FIXME, SNR, signal-to-noise, code review, 注释, 代码审查'
---

# High-SNR Comment Reviewer

## Overview
Code is the best comment — comments must NOT translate code. They explain WHY (intent, tradeoffs, edge cases, design), never WHAT. Test: "Would I delete this in 6 months?" — if it can't stand without the code, it's noise.

## Rules Engine

1. **Anti-Echo** — Delete comments restating code. `// increment counter` before `counter += 1` is zero-value. Rust's `Result`/`Option`/iterators/pattern matching make much "what" self-documenting.

2. **Why, Not What** — Capture what code CANNOT express: business rationale, algorithmic tradeoffs, edge cases, performance decisions, intentional deviations.

3. **No Archeology** — Delete commented-out code; git history preserves it. (`#[cfg(test)]` modules are not commented-out code — they compile and run.)

4. **Drift Detection** — Compare comments with the code whenever either changes; age alone is not evidence. Outdated comments are worse than none.

5. **Doc vs Inline** — Two channels, different jobs:
   - `///` on **public items** (`pub fn`/`struct`/`trait`) = the *contract*. Standard sections: `# Arguments`, `# Returns`, `# Errors`, `# Panics`. Renders in `cargo doc`.
   - `//` inside function bodies = *local why* (non-obvious branch, magic number, workaround). Don't duplicate a `///` doc.
   - Call-site reader needs it → `///`; body maintainer only → `//`.

## Signal Comments (keep)

| Type | Example |
|---|---|
| Design rationale | `// Quicksort over mergesort: in-place, better cache locality for <10K elements` |
| Magic number | `// 0.6 threshold from A/B test Sep 2025; p99 latency dropped 40%` |
| Edge case | `// Empty Vec = "no preferences"; None = "not configured"` |
| Workaround/debt | `// HACK: skip TLS verify until internal CA rotation (Q2 2026)` |
| Performance | `// O(N²) intentional: N<50, benchmarked faster than HashMap` |
| SAFETY | `// SAFETY: ptr non-null, points to initialized T, exclusive &mut` (required on every `unsafe`) |
| API contract (`///`) | `/// Returns Err(Expired) if token TTL < 5min; handle before RPC` |

## Noise Comments (delete)

| Anti-Pattern | Fix |
|---|---|
| `// Get user by ID` on `fn get_user_by_id` | Delete — name says it |
| `// Loop through orders` before `for o in &orders` | Delete |
| `// TODO: fix this` no context/ticket | Add ticket + owner + deadline, or delete |
| Commented-out code | Delete — git preserves it |
| `// Created by Alice 2023-05-12` | Delete — `git blame` has it |
| `// Increment counter by one` | Delete |
| `/***** BEGIN: Payment Processing *****/` | Extract a function instead |
| `// Note: this is important` | Explain WHY, or delete |
| `///` duplicating signature | Document the contract, or delete |

## Workflow
1. Delete comments paraphrasing the next line
2. TODO/FIXME without context — add or delete
3. Commented-out code — delete
4. Each surviving comment: answers "why" not "what"? Rewrite or delete
5. Contradicts the code it annotates? — fix or delete
6. Output cleaned code with deletion justifications
