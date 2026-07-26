---
name: msbuild-antipatterns
description: "Review and fix anti-patterns in .NET 10 SDK-style .csproj, .props, .targets, and Directory.Build files, including broken conditions, duplicated properties, opaque Exec commands, bad item operations, and non-incremental targets. Do not use for non-MSBuild systems or unrelated project work."
license: MIT
---

# MSBuild Anti-Patterns

Use this skill for a focused review of project and build files. It is a
curated adaptation of the MSBuild catalog in `dotnet/skills`, organized around
symptom, impact, and a concrete correction.

## Core question

Does the build graph remain portable, incremental, composable, and explicit as
the repository grows?

## Scope boundary

- Review `.csproj`, `.props`, `.targets`, `Directory.Build.props`,
  `Directory.Build.targets`, and `Directory.Packages.props` files in .NET 10
  projects.
- Use a performance skill when the build has already been measured and the
  issue is evaluation, parallelism, or target timing.
- Do not apply these rules to npm, Maven, CMake, or unrelated XML files.

## Review catalog

| Smell | Risk | Preferred correction |
|---|---|---|
| `<Exec>` for copy, delete, or mkdir | Opaque, non-portable, weak incremental behavior | Use `Copy`, `Delete`, `MakeDir`, `Move`, or `WriteLinesToFile`. |
| Unquoted `Condition` comparisons | Empty values and spaces change evaluation | Quote both operands: `Condition="'$(Configuration)' == 'Release'"`. |
| Absolute machine paths | Breaks on other machines and CI | Use `MSBuildThisFileDirectory`, `MSBuildProjectDirectory`, or a repo-root property. |
| Repeating SDK defaults | Adds noise and pins behavior accidentally | Keep only intentional overrides. |
| Manual `<Compile Include>` in SDK projects | Duplicates implicit globs and misses new files | Remove it or use narrow `Remove`/`Exclude`. |
| Analyzer/tool package without `PrivateAssets="all"` | Build-only dependencies leak to consumers | Mark build analyzers and tools private. |
| Same property block in many projects | Settings drift and maintenance cost | Centralize shared settings in the nearest `Directory.Build.props`. |
| Package versions in every project | Version drift and conflicts | Use Central Package Management where the repo supports it. |
| Target with many unrelated actions | Cannot reason about, skip, or incrementally rebuild steps | Split by responsibility and give each target correct dependencies. |
| Custom target without `Inputs`/`Outputs` | Work repeats on every build | Declare inputs/outputs and register generated files. |
| Replacing `CompileDependsOn` or a similar chain | SDK targets silently stop running | Extend the property: `$(CompileDependsOn);MyTarget`. |
| Import without an existence guard | Fresh clones fail when an optional file is absent | Use `Condition="Exists('...')"` for optional imports. |
| `Include` used where `Update` was intended | Duplicate items or lost metadata | Use `Update` for SDK-globbed items and `Remove` to opt out. |

## Procedure

1. Confirm the project targets `net10.0`, then identify the SDK-style files and
   shared props/targets that participate in evaluation.
2. Read the full relevant file before proposing a change. Check whether a
   property or item is set earlier or later in the import graph.
3. Report findings with file/line locations, the observed symptom, build or
   portability impact, and the smallest concrete fix.
4. For target changes, inspect `DependsOnTargets`, `BeforeTargets`,
   `AfterTargets`, `Inputs`, `Outputs`, and `FileWrites` together. A local fix
   that breaks ordering or cleaning is not a fix.
5. Validate with the narrowest useful command, then run a no-op rebuild to
   confirm the intended work is skipped.

## Path note

Existing backslashes in MSBuild paths may work on .NET 10. Prefer forward
slashes for consistency, but do not report them as a build failure without
evidence.

## Review output

Order findings by correctness, portability, and incremental-build impact. Group
repeated instances of the same pattern, include exact counts, and distinguish
required fixes from optional cleanup. End with a short validation plan rather
than claiming a build is fixed without running it.
