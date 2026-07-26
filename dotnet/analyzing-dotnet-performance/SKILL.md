---
name: analyzing-dotnet-performance
description: "Audit .NET 10 C# code for measurable performance risks across async, allocations, strings, collections, LINQ, regex, serialization, and I/O. Use for hot-path review and optimization triage; do not use for algorithmic complexity, generic code review, or optimization without performance context."
license: MIT
---

# .NET Performance Analysis

Use this skill to find actionable .NET performance risks and prioritize them
without turning every idiom into an optimization. It is a compact adaptation
of the pattern catalog in `dotnet/skills`.

## Core question

Which observed code patterns are likely to affect a measured hot path, and what
smallest change should be benchmarked before adoption?

## Inputs and boundary

Gather the source, known hot paths, workload shape, and any existing profile or
benchmark. Assume the code targets .NET 10. Use this skill for API and allocation patterns;
use `golang-benchmark` only for Go measurement and an algorithm-focused skill
for complexity or data-structure changes. Do not recommend micro-optimizations
for code with no performance requirement.

## Workflow

1. Establish a baseline from an existing benchmark, production profile, or
   clearly stated latency/throughput/allocation target. If none exists, say
   that impact is unverified.
2. Read small files fully. For larger codebases, select topic scans from code
   signals rather than applying every rule everywhere.
3. Check relevant categories: async/task behavior, memory and strings,
   collections/LINQ, regex, serialization/I/O, and structural overhead.
4. Validate suspected hot paths with BenchmarkDotNet or a representative
   profiler. Keep before/after workload, runtime, and configuration identical.
5. Report exact locations and counts, separate evidence from hypotheses, and
   rank findings by measured impact and confidence.

## High-signal patterns

### Async and task usage

- Look for synchronous blocking on `Task`, accidental serial awaits in a loop,
  unnecessary task allocation, and missing cancellation propagation.
- Do not recommend `ValueTask` or `ConfigureAwait(false)` universally. Require
  frequent synchronous completion for `ValueTask`, and library-boundary reason
  for `ConfigureAwait(false)`.
- Treat deadlocks and unbounded concurrency as correctness findings before
  performance findings.

### Allocations, strings, and collections

- Check repeated `Substring`, chained `Replace`, culture-sensitive
  `ToLower`/`ToUpper`, interpolation/concatenation in loops, and avoidable
  `params` arrays on hot paths.
- Check `ToList`/`ToArray`, repeated enumeration, LINQ inside tight loops, and
  per-call collection construction. Preserve readability when the path is not
  measured.
- Suggest `FrozenDictionary`, pooling, spans, or `CollectionsMarshal` only
  when ownership, mutation, safety, and benchmark evidence justify the added
  complexity. Count compound costs across chained calls and delegated methods.

### Regex, serialization, and I/O

- Prefer `[GeneratedRegex]` for compile-time literal patterns in .NET 10; do
  not apply it to dynamic patterns.
- Reuse serializers and clients according to their documented lifetime. Avoid
  creating `HttpClient`, regexes, streams, or serializers per request when the
  application has an established reusable boundary.
- Consider source-generated JSON metadata when serialization overhead is a
  measured requirement; do not change wire behavior casually.

## Severity

- **Critical**: correctness failures, deadlocks, crashes, or a measured severe
  regression.
- **Moderate**: a credible improvement on a known hot path or a repeated
  allocation/concurrency issue with supporting evidence.
- **Info**: a possible improvement whose impact depends on workload or
  profiling.

If hot-path context is unknown, report critical issues normally and label other
findings as conditional. Never present a possible optimization as a measured
result.

## Finding format

For each issue include:

```text
ID. Short title (count)
Impact: one sentence tied to the workload or evidence
Location: File.cs:line
Fix: concrete change to benchmark
Confidence: measured, strongly indicated, or speculative
```

End with the baseline, commands/profiler used, and remaining test or benchmark
gaps. Keep positive patterns brief and do not emit a long catalog of harmless
idioms.
