# Research Notes

This reference records the source-backed rationale for `vcs-naming`. It separates standards from team recommendations so local rules do not get presented as universal requirements.

## Commit Message Sources

- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) defines `<type>[optional scope]: <description>`, optional body and footers, `feat`, `fix`, and breaking-change markers. It allows additional types but does not require a single universal type list.
- [Pro Git, Commit Guidelines](https://git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project) recommends a short subject, a blank line before the body, imperative wording, and a body that explains motivation and context.
- [Git's SubmittingPatches guide](https://github.com/git/git/blob/master/Documentation/SubmittingPatches) recommends logically separate commits, a short subject, an `area: description` prefix, no final period, imperative wording, and a body explaining the problem and rationale.
- [Angular's commit message guidelines](https://github.com/angular/angular/blob/main/contributing-docs/commit-message-guidelines.md) show a stricter Conventional Commits policy with an allowlisted type and scope vocabulary. Treat that strictness as a project choice, not as part of the base specification.

The default policy follows Git and jj's area-prefix style because it gives reviewers useful domain context without requiring semantic-release tooling. Choose Conventional Commits instead when changelog generation, SemVer inference, or commit-parser tooling is a real requirement. Do not silently mix both formats.

## Branch and Bookmark Sources

- [Atlassian branching strategies](https://www.atlassian.com/agile/software-development/branching) describes task/issue branching and the value of putting an issue key in the branch name.
- [Bitbucket branching models](https://confluence.atlassian.com/spaces/BITBUCKETSERVER100/pages/1680278295/Branches) documents common prefixes such as `feature/`, `release/`, `bugfix/`, and `hotfix/`. These are common workflow labels, not a universal Git standard.
- [GitHub rulesets](https://docs.github.com/en/enterprise-server@3.21/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository) support branch-name and commit-metadata restrictions using patterns or regular expressions, subject to GitHub plan and server-version availability.
- [GitLab push rules](https://docs.gitlab.com/user/project/repository/push_rules/) support regular-expression validation for commit messages and branch names.

The naming pattern in the skill is therefore a recommended team contract: a workflow kind, optional issue identity, and a short lowercase slug. Lowercase ASCII slugs avoid case-sensitive duplicates and filesystem portability problems; an uppercase issue key is allowed when the tracker requires it.

## Jujutsu Sources

- [jj contributing guidelines](https://jj-vcs.github.io/jj/latest/contributing/) say that jj does not use Conventional Commits. Its commits start with `<topic>:`; each commit should ideally do one thing; tests and documentation belong with the code they cover; and contributors should update the relevant commit instead of appending review-fix commits.
- [jj bookmarks](https://jj-vcs.github.io/jj/latest/bookmarks/) defines bookmarks as named pointers analogous to Git branches and explains that a bookmark pushed through Git becomes a branch with the same name. jj has no active/current bookmark concept.
- [jj working copy and workspaces](https://jj-vcs.github.io/jj/latest/working-copy/) defines a workspace as a working copy plus its linked `.jj` directory and documents multiple workspaces backed by one repository. It does not prescribe semantic workspace names.
- [jj Git compatibility](https://jj-vcs.github.io/jj/latest/git-compatibility/) documents the mapping between jj bookmarks and Git branches and currently lists Git hooks as unsupported.
- [jj workspace add implementation](https://github.com/jj-vcs/jj/blob/main/cli/src/commands/workspace/add.rs) documents the `--name` option and the default behavior of using the destination directory basename as the workspace name.

The jj-specific consequence is that bookmark names are durable collaboration names, while workspace names are local working-copy identities. A local anonymous working-copy commit or temporary description is normal and should not be subjected to the same rule as an outgoing bookmark or published revision.

## Boundary With Other Skills

- `vcs-router` owns backend detection before VCS commands.
- `jujutsu` owns safe status, diff, graph, rewrite, bookmark, and workspace operations after detection.
- `jujutsu-parallel` owns fixed-base multi-agent workspaces, ownership, and handoff coordination.
- `vcs-naming` owns the naming contract and reviews of names across Git and Jujutsu.
