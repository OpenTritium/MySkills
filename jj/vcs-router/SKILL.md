---
name: vcs-router
description: "Detect the current repository's version-control backend before any status, diff, log, mutation, or workspace operation, then route to Git or Jujutsu workflows. Use when a task touches repository history, branches, worktrees, bookmarks, commits, or conflicts, especially in repositories that may contain `.jj`, `.git`, or both. 中文触发：版本控制检测、Git 还是 jj、仓库后端、VCS 路由。"
---

# VCS Backend Router

## Purpose

Identify the repository backend before running a VCS command. Return one backend—`vcs=jj`, `vcs=git`, `vcs=broken-jj`, or `vcs=none`—and load only the workflow that matches it.

## Detection

1. Start at the current directory and walk upward to the nearest ancestor containing `.jj` or `.git`; a `.git` file counts as a Git worktree marker.
2. If `.jj` is present, run `jj root`. On success select `vcs=jj`. On failure select `vcs=broken-jj`; do not fall back to Git, even when `.git` is also present.
3. If `.jj` is absent, run `git rev-parse --show-toplevel`. On success select `vcs=git`; on failure select `vcs=none`.

When `.jj` and `.git` coexist, Jujutsu is authoritative after `jj root` succeeds. Do not use `status`, `diff`, or a mutation as a backend probe; the probe must finish first.

## Route

| Result | Next action |
|---|---|
| `vcs=jj` | Use `jujutsu`; use `jujutsu-parallel` only for parallel Jujutsu workspaces |
| `vcs=git` | Use Git-native commands and Git worktrees; do not invoke Jujutsu skills |
| `vcs=broken-jj` | Report the broken Jujutsu environment and stop; do not mutate through Git |
| `vcs=none` | Treat the directory as unmanaged; do not initialize a repository unless requested |

Pass the selected backend and repository root to review or implementation skills that need VCS context. For a broad refactor, use the backend-specific command group only after routing.

## Command Discipline

- Run the probe once at task start and reuse the result unless the working directory changes.
- After selecting Git, use `git --no-pager status`, `git --no-pager diff`, and other Git commands only.
- After selecting Jujutsu, use `jj --no-pager status`, `jj --no-pager diff --git`, and the `jujutsu` workflow only.
- Keep backend detection read-only. Never initialize, convert, fetch, push, commit, or create a workspace as part of detection.
- Before a mutation or push, re-check that the selected root and backend still match the task context.

## Handoff

State `vcs`, repository root, whether `.jj` and `.git` coexist, and the selected follow-up skill. If detection fails, report the exact failed probe and the safe next step instead of guessing.
