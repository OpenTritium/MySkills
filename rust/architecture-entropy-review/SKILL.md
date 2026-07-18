---
name: architecture-entropy-review
description: "Review large refactors, especially AI-generated Rust changes, for architectural entropy: duplicate semantic implementations, split sources of truth, divergent execution routes, domain rules leaking into outer layers, abstraction boundaries being bypassed or widened, migration leftovers, dependency-direction drift, and tests that protect implementation details instead of contracts. Use for broad refactors, module moves, domain redesigns, API migrations, new service layers, compatibility paths, or changes described as cleanup, unification, simplification, or making the code compile. Chinese triggers include 架构熵增、重构后审查、重复实现、路线偏离、领域破坏、抽象泄漏、边界腐化、事实来源分裂、AI 重构 review、架构漂移。"
---

# Architecture Entropy Review

## Scope

Review large refactors for changes that increase the number of semantic owners, representations, execution routes, exceptions, or dependencies without a compensating contract or boundary.

Treat “entropy” as a proof obligation, not as a synonym for long code or a different design preference. Report only concrete architectural drift. This is a review skill: do not edit code unless the user separately asks for implementation.

## Workflow

### 1. Establish the intended route

Read the diff, task, design note, and nearby history before judging the architecture. Map:

- the canonical entry point for each changed behavior;
- the type or module that owns each domain invariant;
- the source of truth for state and configuration;
- the intended dependency direction;
- old paths that should be removed, retained for compatibility, or coexist deliberately.

If no design document exists, infer the route from repeated existing usage and label that inference. Do not report a preference as a defect without showing an inconsistency.

### 2. Trace the before/after shape

Use the smallest useful search surface. First apply `vcs-router`, then use only the command group matching its result:

    Git: git --no-pager diff --stat
         git --no-pager diff
    JJ:  jj --no-pager diff --git
    rg 'old_symbol|new_symbol|domain_term|feature_flag'

Then trace changed symbols from callers to callees, inspect public exports and visibility, and compare production paths with tests and fixtures. For Rust, use module declarations, pub use, visibility, trait implementations, compiler diagnostics, LSP references, and focused cargo checks/tests when available.

Keep a short ledger:

| Concern | Before | After | Evidence |
|---|---|---|---|
| Entry point | symbol/path | symbol/path(s) | callers and exports |
| Rule owner | type/module | type/module(s) | validation and mutation |
| State representation | type/field | type/field(s) | reads, writes, conversion |
| Dependency direction | A -> B | A -> B, C | imports and calls |
| Migration code | expected lifetime | active/dead path | references and flags |

### 3. Prove entropy signals

Treat these as search hypotheses. Promote one to a finding only when it connects to a duplicated rule, weakened boundary, split ownership, or an additional future-change path.

| Signal | Inspect | Required proof |
|---|---|---|
| Semantic duplication | validation, normalization, authorization, defaulting, retry, timeout, pagination, error mapping, policy tables, DTO conversions | one business rule has multiple owners or can diverge |
| Route bypass | handlers, services, repositories, adapters, read paths, cache fast paths | a new caller skips an existing invariant, authorization, transaction, error policy, instrumentation, or lifecycle |
| Domain erosion | raw strings/numbers/bools/maps, public fields, generic context bags, infrastructure types | a named invariant or domain term lost its owner |
| Abstraction escape | widened visibility, re-exports, downcasts, concrete construction, hooks, flags, callbacks, one-use wrappers | an abstraction no longer hides its dependency or adds no real contract |
| Dependency/ownership drift | imports, re-exports, globals, shared utilities, state mutation, lifecycle transitions | a new edge or owner reverses direction or fragments state/cleanup |
| Migration residue | aliases, shims, dual writes, flags, TODO branches, dead exports, old tests | reachable parallel mechanisms lack a narrow boundary or removal condition |
| Accidental generality | registries, plugin systems, strategy hierarchies, generic policies, configuration layers | abstraction serves one case and protects no existing variation point |
| Test/contract drift | old-path tests, internal construction, duplicate fixtures, compile-only checks, snapshots | tests do not exercise the route or invariant now used in production |

For each candidate, ask:

    When this rule changes, how many places must a maintainer know to update?

Do not flag intentional DTO/persistence/read-model translations, anti-corruption layers, distinct policies, compatibility code with a named consumer and exit condition, or performance-local duplication when the semantic contract remains centralized.

### 4. Validate and classify

Before writing a finding, identify one concrete caller-to-behavior path and at least one of:

- a second implementation or representation;
- a bypassed validation, invariant, side effect, or lifecycle rule;
- a widened dependency or visibility edge;
- a reachable migration branch without an exit condition;
- a test that proves the new route does not protect the contract.

When a refactor or bug report leaves the behavior contract uncertain, require focused happy/unhappy characterization or regression tests through the stable boundary before endorsing a new owner or route. Use `testing-strategy` for the test design; do not infer intended behavior from the new structure.

Use severity by impact:

- P1: normal operation can produce incorrect domain behavior, duplicate side effects, unauthorized state changes, or split-source-of-truth behavior.
- P2: a reachable divergent route, leaked invariant, uncontained migration path, or boundary erosion will predictably cause inconsistent future changes.
- P3: a local but real increase such as dead residue, a semantic-empty wrapper, unnecessary generality, or limited test drift.

State confidence separately: high for a reachable path or duplicate rule, medium for strong structural evidence, and low for a hypothesis needing runtime or product confirmation. Never raise severity to compensate for low confidence.

Run only relevant checks. For Rust, prefer formatting/checking and focused tests around changed modules, then broader tests when crate or public API boundaries changed. A clean build proves compilation, not architectural integrity.

## Reporting

Report findings first, ordered by severity and then confidence. Use one finding per independent problem:

    [P1 | high] Authorization now has two reachable owners

    Evidence: path/to/new.rs:42 calls storage directly; path/to/use_case.rs:18
    performs authorization on the established route. The new route is reachable
    from path/to/handler.rs:27 and skips that check.

    Why this increases entropy: the same operation has two owners, so the next
    policy change must update both paths; the current divergence is a bug.

    Smallest repair: route the handler through use_case::execute, or declare
    the new path as an explicit projection with its own contract.

    Validation: add a boundary test through both entry points and run focused
    module tests.

Use file and line references. Explain the violated invariant or boundary, the future-change cost, and the smallest safe repair. If no candidate meets the evidence bar, say so explicitly, then list residual risk or missing validation and summarize the reviewed scope. Do not replace findings with “consider refactoring”.

## Boundary With Other Skills

Use this skill when a local smell has architectural consequences. Defer purely local concerns to the narrower skill: function shape to func-smell, module/type structure to rust-structure-refactor, type invariants to encode-invariant, dependency integration to rust-ecosystem, and tests to testing-strategy. Reuse their evidence when it proves that a route, owner, or boundary multiplied.
