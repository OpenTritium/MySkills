---
name: jujutsu-parallel
description: "Coordinate multiple agents working in parallel on a Jujutsu repository after the backend has been detected. Use with jujutsu when several agents, processes, or workspaces need isolated changes, fixed-base development, unique bookmarks, handoff, integration, stale-workspace recovery, or conflict-safe stacked-diff workflows. Covers multi-agent development, parallel coding, workspace isolation, divergent changes, and Jujutsu collaboration."
allowed-tools: Bash(jj *)
---

# Jujutsu Parallel Agents

Use this skill together with jujutsu. It turns Jujutsu's safe concurrent-edit model into an operational protocol that minimizes stale workspaces, overlapping edits, and accidental history rewrites.

## Activation Boundary

Run `vcs-router` first, then use `jujutsu` only when it returns `vcs=jj`. A Git repository is outside this skill's scope. Do not run any `jj` command for a Git-only repository; use Git-native worktrees instead.

## Official Model

- A workspace is a working copy plus a linked .jj directory backed by the same repository. Each workspace has its own working-copy commit.
- Jujutsu is designed to make concurrent repository modifications safe, and concurrent edits to a change are allowed. Safe does not mean conflict-free: rewriting the commit used as another workspace's @ makes that workspace stale.
- The commit graph is the source of truth. Change IDs remain stable across rewrites, descendants may be auto-rebased, and conflicts can remain recorded in commits until someone resolves them.
- Use native jj workspaces rather than Git worktrees for multiple working copies backed by one repo.

## Protocol

### 1. Freeze the base and ownership

Before any agent edits:

1. Choose one fixed base revision. Resolve a moving bookmark to a change ID and record both the ID and the base description.
2. Assign each agent a responsibility and preferred file set.
3. Assign an integration owner.
4. Serialize changes to shared files: workspace manifests, lockfiles, generated output, schemas, migrations, public export lists, and repository-wide formatting.

Inspect the base:

       jj --no-pager log -r <base-change-id>
       jj st

Do not let agents independently start from moving @ or from a bookmark that another agent may move.

### 2. Create one workspace per agent

Create an independent child revision for each agent:

       jj workspace add --name agent-a -r <base-change-id> -m "Agent A: implement parser" ../agent-a
       jj workspace add --name agent-b -r <base-change-id> -m "Agent B: add API tests" ../agent-b
       jj --no-pager workspace list -T builtin_workspace_list_with_root

The -r option makes the new workspace's working-copy commit a child of the fixed base. Do not use jj edit <base-change-id> in multiple workspaces; that intentionally shares the same @ and makes concurrent rewrites harder to coordinate.

Create unique bookmarks only when a durable handoff name is useful. Prefixing names by agent or topic makes ownership visible:

       jj bookmark create agent-a -r @
       jj bookmark create agent-b -r @

Never have parallel agents move the same release bookmark.

### 3. Keep agent changes local

- Keep each agent's work in its own child revision or stack. Use jj desc, jj new, and jj commit to refine it.
- Do not run jj squash, jj absorb, or jj rebase against the shared base or another agent's active revision. These rewrite graph nodes globally and can stale peer workspaces or auto-rebase their descendants.
- Do not modify the same file or generated artifact concurrently. If overlap is unavoidable, make it an explicit integration task.
- Do not push from individual agents unless explicitly requested.
- Run tests from the agent workspace and record the command, result, and base change ID.

Before handoff, each agent must run:

       jj st
       jj --no-pager diff --git
       jj --no-pager show @

Report the change ID or unique bookmark, changed paths, tests, unresolved conflicts, and assumptions.

### 4. Integrate in a dedicated workspace

Create a separate integration workspace from the same base:

       jj workspace add --name integration -r <base-change-id> -m "Integrate parallel work" ../integration
       jj --no-pager log -r 'agent-a | agent-b'

Keep source workspaces untouched while integrating.

Prefer an intentional merge revision while source agents remain active:

       jj new <agent-a-change-id> <agent-b-change-id>

This preserves both source revisions and exposes overlapping changes as a conflict in the integration workspace. Jujutsu can record that conflict and let the integration owner resolve it later.

If a linear history is required, duplicate before moving content so the source revision remains untouched:

       jj duplicate --onto <agent-a-change-id> <agent-b-change-id>
       jj --no-pager log -r 'agent-a-change-id::'

Inspect the new duplicate, edit it in the integration workspace, and test it. Use jj rebase -r or jj rebase -s on the original agent revision only after that agent workspace is explicitly frozen or disposable; rebase rewrites the selected revision globally.

### 5. Resolve and validate

Resolve conflicts only in the integration workspace:

1. Inspect jj st and the conflicted paths.
2. Create a child resolution revision with jj new <conflicted-change-id>, or edit the current integration working copy when appropriate.
3. Edit conflict files and remove markers; use jj resolve only when an approved non-interactive merge tool is available.
4. Run jj st and focused tests.
5. Run jj squash to fold the resolution into the integration conflict revision only after reviewing the result.

After all agents are integrated:

       jj st
       jj --no-pager diff --git
       jj --no-pager log -r 'base-change-id::@'

Run tests for each changed boundary, then broader checks when shared APIs, generated files, or dependency manifests changed. Move the release bookmark only after validation and explicit approval.

### 6. Recover stale workspaces

When another workspace rewrites a revision used as your @:

1. Run jj st and identify whether the working copy is stale or contains unsnapshotted edits.
2. Preserve and describe any local work before updating.
3. Run jj workspace update-stale.
4. Inspect jj log for a divergent change ID or recovery commit.
5. Compare the recovered content with the agent's handoff before continuing or abandoning anything.

Never abandon a divergent commit merely to make status clean. Identify its owner and diff first.

When an agent is done, run jj workspace forget <workspace-name>; delete its files separately only after confirming they are no longer needed.

## Anti-Patterns

| Anti-pattern | Consequence | Correction |
|---|---|---|
| Multiple agents jj edit the same @ | stale or divergent working copies | create independent child revisions with workspace add -r |
| All agents move one bookmark | bookmark conflicts and unclear ownership | use unique agent bookmarks; move release bookmark during integration |
| Agent squashes into shared base | rewrites peers' parent and auto-rebases descendants | keep a private stack until integration |
| Concurrent lockfile/generated-file edits | noisy conflicts and non-deterministic output | assign one integration owner |
| Linearize by rebasing an active peer revision | peer workspace becomes stale | merge revisions or duplicate first |
| Delete workspace directory directly | repository retains a workspace record | run workspace forget first |
| Resolve conflicts in a source workspace | peer sees rewritten or partially resolved work | resolve in integration workspace |
