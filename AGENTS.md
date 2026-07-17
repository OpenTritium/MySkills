# MySkills Contributor Guide

## Repository Layout

- `rust/` contains Rust language, design, review, and tooling skills.
- `jj/` contains Jujutsu version-control skills.
- Keep existing skill paths stable unless a migration alias or explicit compatibility plan exists.

## Skill Contract

- Give each skill one primary responsibility and a precise frontmatter description.
- State the skill's core question, decision boundary, and related skills when overlap is possible.
- Keep `SKILL.md` concise; move genuinely conditional detail into a directly linked `references/` or `examples/` file.
- Add or update the relevant case in `test-triggers.md` when changing triggers or boundaries.
- Update the categorized skill index in `README.md` when adding or renaming a skill.

## Boundary Rules

- Use `architecture-entropy-review` for cross-module or cross-route architectural drift.
- Use `rust-structure-refactor` for local function, struct, and module decomposition.
- Use narrower skills for naming, errors, tests, async behavior, resources, and type invariants.
- Do not introduce a second skill with the same primary trigger without documenting the routing boundary.

## Validation

Run the skill validator for every changed or added skill:

```bash
python3 /home/tritium/.codex/skills/.system/skill-creator/scripts/quick_validate.py rust/<skill-name>
```

Check frontmatter, README membership, trigger ownership, related-skill boundaries, and template residue before handoff.
