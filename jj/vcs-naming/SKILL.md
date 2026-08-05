---
name: vcs-naming
description: "Choose or review naming conventions for Git commits, branches, Jujutsu bookmarks, and workspaces. Use when standardizing commit messages, branch/bookmark names, workspace names, or repository contribution rules. Do not use for executing VCS operations, detecting the backend, or coordinating parallel workspaces. 中文触发：提交消息、分支命名、书签命名、工作空间命名、命名规范、提交规范"
---

# VCS Naming Conventions

Use this skill to make names readable, searchable, and enforceable across Git and Jujutsu. The default policy below follows the Jujutsu project's `<area>: <description>` style while preserving Conventional Commits as an explicit alternative when release automation requires it.

## When to Use

- Choose or review a commit message format.
- Choose or review a Git branch or Jujutsu bookmark name.
- Choose or review a Jujutsu workspace name.
- Write or audit repository contribution rules for these names.
- Decide where local WIP state ends and publishable naming rules begin.

## Decision Boundary

- Use `vcs-router` before running repository status, diff, log, mutation, or workspace commands.
- Use `jujutsu` for Jujutsu operations after backend detection.
- Use `jujutsu-parallel` for multi-agent or concurrent workspace coordination.
- Use this skill for the naming contract itself, regardless of whether the backend is Git or Jujutsu.

## Commit Messages

Prefer this format unless the repository explicitly requires Conventional Commits:

```text
<area>: <imperative description>

<why, behavior change, trade-offs, and tests>

<optional trailers>
```

Rules:

1. Use a stable area, module, command, or responsibility as `<area>`.
2. Use an imperative verb: `add`, `fix`, `remove`, `clarify`, or `document`.
3. Keep the subject under 50 characters when practical and under 72 characters at the limit.
4. Do not end the subject with a period. Use lowercase after the area prefix.
5. Add a body when the reason or behavioral impact is not obvious. Explain why and what changed; do not narrate the diff.
6. Keep one logical change per revision. Keep directly related tests and documentation with that change.
7. Use trailers consistently, such as `Refs: #123`, `Fixes: #123`, and `BREAKING CHANGE: ...`.

Examples:

```text
docs: clarify jj workspace naming
rust-structure-refactor: narrow module visibility
ci: validate skill frontmatter
```

Use Conventional Commits only when tooling depends on semantic types or release automation:

```text
<type>(<scope>): <description>
```

Do not mix `feat:`/`fix:` semantics with the area-prefix policy without an explicit repository rule.

## Branches and Bookmarks

Treat a bookmark as the Jujutsu equivalent of a Git branch and as the durable name for a shared review or release topic:

```text
<kind>/<ticket>-<short-slug>
```

Recommended kinds are `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`, `perf`, `release`, and `hotfix`.

Examples:

```text
feat/PROJ-123-add-jj-validation
fix/PROJ-456-handle-stale-workspace
docs/update-contributing
```

Use short ASCII names with lowercase slugs, digits, hyphens, and `/`. Preserve an uppercase issue key only when the issue tracker requires it. Avoid spaces, non-ASCII text, dates, usernames, random hashes, and publishable names such as `wip` or `tmp`. Keep `main` or the repository's existing trunk name for the integration line; do not add `develop` without a real workflow need.

## Jujutsu Workspaces

A workspace name identifies a local working copy, not a remote branch. Use:

```text
<ticket>-<short-slug>[-<purpose>]
```

Examples:

```text
PROJ-123-add-jj-validation
PROJ-123-add-jj-validation-test
PROJ-123-add-jj-validation-review
PROJ-123-add-jj-validation-agent-1
```

Use a purpose suffix such as `dev`, `test`, `review`, `repro`, or `agent-1` when several workspaces serve the same topic. Prefer a sibling path such as `../ws/<workspace-name>`. Keep the primary workspace's existing basename or use `primary`; do not use `main` when that would confuse it with the main bookmark.

## Jujutsu Workflow Implications

- An unnamed or temporary working-copy commit is normal during local jj work; enforce the convention on described or outgoing revisions.
- Create a bookmark only when a durable handoff or remote push name is useful.
- Before publishing, use `jj describe`, `jj split`, and `jj squash` to make each revision atomic and properly described.
- A bookmark name and workspace name may share a base slug, but they represent different resources.

## Enforcement

Prefer CI or server-side validation for jj repositories. The jj Git-compatibility documentation lists Git hooks as unsupported, and jj rewrites mutable descriptions during normal work. Validate outgoing revisions and bookmark names at the push or review boundary instead of rejecting every temporary working-copy state.

## Related Skills

- `vcs-router`: detect Git versus Jujutsu before repository operations.
- `jujutsu`: execute and verify Jujutsu commands.
- `jujutsu-parallel`: coordinate isolated workspaces and unique handoff names.
- [`references/research.md`](./references/research.md): source-backed rationale and standards comparison.

## Verification

Before adopting or enforcing a rule, verify:

- One unambiguous primary format is documented.
- Commit, bookmark/branch, and workspace names have separate contracts.
- Temporary local jj state is not confused with publishable history.
- The chosen pattern can be checked by the available CI or forge tooling.
- The repository index and trigger matrix route naming questions to this skill.
