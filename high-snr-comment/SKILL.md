---
name: high-snr-comment
description: Use when reviewing or writing code comments — eliminating redundant comments, code-translating comments, outdated comments, commented-out code, empty cheerleading. Keywords: comment, annotation, doc, Javadoc, godoc, docstring, ///, //, /*, TODO, FIXME, SNR, signal-to-noise, code review, 注释, 代码审查
---

# High-SNR Comment Reviewer

## Overview
Code is the best comment — comments must NOT translate code. Comments explain WHY (intent, tradeoffs, edge cases, design decisions), never WHAT (the code already says that). Every comment must pass the "Would I delete this in 6 months?" test: if it can't stand alone without the code, it's noise.

## When to Use
- Reviewing PRs with comment additions or changes
- Cleaning up legacy comments that contradict code
- Auditing codebase for commented-out code blocks
- Establishing team comment conventions

## Rules Engine

1. **Anti-Echo** — Delete comments that merely restate the code. `// increment counter` before `counter++` is zero-value. The code is the spec.

2. **Why, Not What** — Comments must capture information the code CANNOT express: business rationale, algorithmic tradeoffs, known edge cases, performance decisions, intentional deviations from convention.

3. **No Archeology** — Kill commented-out code immediately. Git history preserves it; the codebase must not. Zombie code rots and confuses.

4. **Drift Detection** — Any comment more than 2 PRs old is suspect. Comments and code diverge over time; outdated comments are worse than no comments. When modifying code, update or delete adjacent comments.

## Signal Comments (keep)

| Type | Example |
|---|---|
| Design rationale | `// Quicksort chosen over mergesort: in-place, better cache locality for <10K elements` |
| Magic number | `// 0.6 threshold from A/B test Sep 2025; p99 latency dropped 40%` |
| Non-obvious edge case | `// Empty list is valid here: signals "no preferences" vs nil = "not configured"` |
| Workaround/tech debt | `// HACK: skip TLS verify until internal CA rotation completes (Q2 2026)` |
| Performance note | `// O(N²) is intentional: N<50 per profile constraint, benchmarked faster than map` |
| Public API contract | `// Returns ErrExpired if token TTL < 5min; callers MUST handle this before RPC` |

## Noise Comments (delete)

| Anti-Pattern | Fix |
|---|---|
| `// Get user by ID` on `func GetUserByID(id string)` | Delete — name already says it |
| `// Loop through orders` before `for _, o := range orders` | Delete — code is self-evident |
| `// TODO: fix this` without context or ticket | Delete or add ticket link + owner + deadline |
| Commented-out code block with `//` prefix | Delete — git history preserves originals |
| `// Created by Alice on 2023-05-12` | Delete — git blame provides this |
| `// Increment the counter by one` on `counter += 1` | Delete — zero information value |
| `/***** BEGIN: Payment Processing *****/` | Delete — use function extraction, not banners |
| `// Note: this is important` | Delete — explain WHY it's important |

## Workflow
1. Scan for comments that paraphrase the next line of code — delete them
2. Find comments starting with "TODO" or "FIXME" — add context or delete
3. Spot commented-out code blocks — delete immediately
4. For each surviving comment, verify it answers "why", not "what" — rewrite or delete
5. Cross-check: does any comment contradict the code it annotates? — fix or delete
6. Output cleaned code with a summary of deletions and their justifications
