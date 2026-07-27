---
name: jujutsu
description: "Operate safely on a Jujutsu repository after `vcs-router` detects `vcs=jj`. Use for Jujutsu status, diff, log, commit, selective path splitting, branch, rebase, push, restore, bookmark, workspace, and conflict operations, including mixed working-copy changes. Do not use for Git-only repositories; use `vcs-router` first and `jujutsu-parallel` only for parallel Jujutsu workspaces. 中文触发：Jujutsu、jj、变更、按路径拆分、混合工作区、书签、工作区、冲突、并行工作区。"
allowed-tools:
  - Bash(jj *)
---

# Jujutsu (jj)

Use this skill only after `vcs-router` confirms `vcs=jj`. The workflow is checked against jj v0.43.0; inspect local help when a command differs.

## Mandatory Agent Rules

1. Run `vcs-router` first and continue only when it returns `vcs=jj`.
2. Use only Jujutsu commands; never switch to raw Git workflow commands inside this skill.
3. Use --no-pager for commands that print output:

       jj --no-pager status
       jj --no-pager log
       jj --no-pager diff --git
       jj --no-pager show <change-id>

4. Use -m for descriptions or mutations that would otherwise open an editor:

       jj desc -m "message"
       jj new -m "message"
       jj commit -m "message"
       jj squash -m "message"

5. Before every mutation, inspect `jj st`, the complete working-copy diff, and the intended graph. After every mutation, run `jj st` and a targeted diff or log check; do not batch a chain of rewrites.
6. Avoid interactive commands in an agent environment: `jj split` without explicit filesets, `jj squash -i`, diff editors, and external jj resolve tools. Use the non-interactive split workflow below when exact paths are sufficient.
7. Do not push unless the user explicitly asks. Before pushing, inspect jj status, log, diff, and bookmark state.

## Core Model

The following sections apply only when `vcs=jj`.

- The repository graph is the source of truth; the working copy is a commit, referenced as @.
- All current working-copy changes are in `@`; there is no staging area. `jj commit -m "message"` without filesets selects all changes in `@` and starts a new working-copy revision.
- Most jj commands snapshot the working copy at the beginning, record an operation, and update the working copy afterward.
- Treat `jj commit` as a whole-`@` operation unless explicit filesets are provided; use `jj split` for a deliberate, reviewable path-based extraction.
- Change IDs remain stable when a change is rewritten; commit IDs change. Prefer change IDs when referring to work.
- Revisions are mutable. Conflicts can be recorded in commits, descendants may be automatically rebased, and operations can be inspected or undone.

## Essential Workflow

When `vcs=jj`:

### Start Work

Inspect the current revision before editing:

       jj st

If the working-copy commit contains unrelated work before editing, create a new revision:

       jj new -m "Describe the new change"

Changes made in the working copy are automatically included in the current revision. Use jj desc -m when describing an existing revision.

### Split a Mixed Working-Copy Revision

Use this workflow when `@` already contains multiple logical changes and only a subset should become a separate revision. Decide the final graph before rewriting it:

       base -> selected change -> remaining WIP (@)

Do the following preflight without mutating the repository:

       jj st
       jj --no-pager diff --from @- --to @ --stat
       jj --no-pager diff --from @- --to @ --name-only
       jj --no-pager log -r '@- | @'

Record the current `@` change ID, its parent change ID, and the exact paths to select. A path fileset is cwd-relative by default, so either work from the repository root or use `root:`/`root-file:` explicitly. Quote filesets, especially those containing operators or glob characters. Do not select a directory by a guessed basename when an exact root-relative path is available.

Use this non-interactive form; supplying filesets avoids the diff editor:

       jj split -r @ -m "Selected change" -- \
         'root:src/cdc-engine-pod' \
         'root:tests/cdc-engine-pod'

The selected paths become the lower revision and the unselected paths remain in the new child, which should be the working-copy revision. Immediately verify both sides and the graph:

       jj st
       jj --no-pager log -r '<base-change-id>::@'
       jj --no-pager diff --stat -r <selected-change-id>
       jj --no-pager diff --name-only -r @
       jj --no-pager bookmark list

If the selected and remaining changes overlap in one file, path-based splitting cannot separate hunks safely. Stop and resolve that boundary deliberately before rewriting history. If the graph or path sets differ from the plan, stop; inspect the operation and use `jj undo` only after confirming the latest operation is the mistaken one.

`jj split` normally moves bookmarks from the old revision to the new child. If the bookmark should identify the selected revision instead, move it explicitly only after reviewing the resulting graph.

### Inspect History and Changes

       jj --no-pager log
       jj --no-pager log -r 'trunk()..@'
       jj --no-pager diff --git
       jj --no-pager show <change-id>
       jj --no-pager evolog <change-id>

The default diff is side-by-side; use --git for unified +/- output. Use change IDs for stable references and commit IDs when a specific immutable snapshot is required.

### Refine Changes

Keep one logical change per revision. Use:

       jj desc -m "message"
       jj commit -m "message"
       jj new -m "message"
       jj squash -m "message"
       jj split -r @ -m "message" -- <filesets>
       jj absorb
       jj rebase -r <change-id> -d <destination>
       jj rebase -s <change-id> -o <destination>
       jj abandon <change-id>
       jj undo

Review the result after rewriting:

       jj st
       jj --no-pager show @

Use jj commit when intentionally finalizing the current working-copy revision and starting the next one. Use jj new plus jj squash when you need a reviewable child revision before folding its changes back.

### Run Commands Across Revisions

jj run executes a command in isolated working copies and amends the selected revisions. Treat it as a mutating command:

       jj run -r 'trunk()..@' -- cargo check

Inspect jj st and the resulting diff afterward. Use --clean when build artifacts must not be reused.

### Restore Files

       jj restore
       jj restore path/to/file
       jj restore --from <change-id> path/to/file

Restoring the working copy discards content from the current revision. Confirm the target and run jj st afterward.

## Bookmarks

Bookmarks are named pointers similar to Git branches. Creating a new revision does not advance a bookmark; rewriting a bookmarked revision generally moves it with the change, while abandoning a bookmarked revision deletes the associated bookmark.

       jj --no-pager bookmark list
       jj bookmark create my-feature -r @
       jj bookmark set my-feature -r @
       jj bookmark move my-feature --to <change-id>
       jj bookmark delete my-feature

Before pushing, move or create the intended bookmark explicitly:

       jj bookmark move my-feature --to @
       jj git push -b my-feature

## Workspaces

Workspaces are multiple working copies backed by one repository. Each workspace can have a different working-copy commit:

       jj workspace add ../my-tests
       jj workspace add --name tests -r <change-id> ../my-tests
       jj --no-pager workspace list -T builtin_workspace_list_with_root
       jj workspace root --name <workspace>
       jj workspace forget <workspace>
       jj workspace update-stale

Files are not live-mirrored. Do not edit a change that another workspace uses as @ unless that shared edit is deliberate and coordinated. Use jujutsu-parallel for the safer multi-agent protocol.

## Git Integration

For a non-colocated jj repository, use jj's Git integration:

       jj git clone <url>
       jj git fetch
       jj git fetch --remote <remote-name>
       jj git fetch -b <branch-name>
       jj git push -b <bookmark-name>

When `.jj/` and `.git/` coexist, `vcs-router` selects `vcs=jj`; keep workflow commands in jj. Use `jj git import` or `jj git export` only for the explicit Git integration boundary.

Initialize or convert repositories with:

       jj git init --colocate
       jj git colocation status
       jj git colocation enable
       jj git colocation disable

## Conflicts

Jujutsu can record conflicts in commits, so a rebase or merge can complete before resolution. Resolve in a working copy, inspect the result, and fold the resolution into the conflicted revision:

       jj new <conflicted-change-id>
       jj st
       jj squash

Alternatively edit the conflict files directly in the conflicted working copy. Do not discard a conflicted revision merely because it is not immediately resolvable.

## Quick Reference

| Action | Command |
|---|---|
| Status | jj st |
| Log | jj --no-pager log |
| Diff | jj --no-pager diff --git |
| New revision | jj new -m "message" |
| Describe | jj desc -m "message" |
| Commit all current @ changes and start next | jj commit -m "message" |
| Split selected paths | jj split -r @ -m "message" -- <filesets> |
| Edit revision | jj edit <change-id> |
| Show evolution | jj --no-pager evolog <change-id> |
| Duplicate | jj duplicate <change-id> |
| Squash | jj squash |
| Absorb | jj absorb |
| Rebase | jj rebase -s <change-id> -o <destination> |
| Abandon | jj abandon <change-id> |
| Restore | jj restore [paths] |
| Undo | jj undo |
| Fetch | jj git fetch |
| Push | jj git push -b <bookmark> |
| Add workspace | jj workspace add <path> |
| Update stale workspace | jj workspace update-stale |
| Run across revisions | jj run -r '<revset>' -- <command> |

Before considering a Jujutsu operation complete, inspect `jj st`, its diff, description, evolution, and bookmark state.
