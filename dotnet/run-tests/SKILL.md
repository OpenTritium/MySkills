---
name: run-tests
description: "Choose or run the correct TUnit command for .NET 10 projects, including test filters and project discovery. Do not use for writing test code, test logic debugging, CI configuration, or unrelated test tooling."
license: MIT
---

# Run TUnit Tests

Use this skill whenever the user asks to run, filter, or troubleshoot a TUnit
command. Detect the project layout and SDK version before proposing flags, then
use TUnit's own command-line filter syntax.

## Core question

What exact command runs the requested TUnit scope in the project's .NET 10
test project?

## Boundary

- Use this skill for TUnit discovery, command construction, filtering, and
  execution.
- Use `writing-tunit-tests` for writing or fixing TUnit source and assertions.
- Use `golang-testing` or `rust-testing-strategy` for Go or Rust test behavior.
- Do not use this skill to debug the production logic behind a failing test or
  to change test tooling.

## Detection order

Inspect these files before choosing a command:

1. `global.json` to confirm the repository selects a .NET 10 SDK.
2. `.csproj` for the `TUnit` package, `net10.0` target, output type, and test
   project settings.
3. `Directory.Build.props` and `Directory.Packages.props` for shared settings
   and centrally managed package versions.

Confirm the local SDK with `dotnet --version` and stop if it is not a .NET 10
SDK. Do not add commands for other target versions.

## Command rules

Run a TUnit project as an executable through `dotnet run`:

```bash
# All tests in a project
dotnet run --project tests/App.Tests

# One class
dotnet run --project tests/App.Tests -- --treenode-filter "/*/*/LoginTests/*"

# One category
dotnet run --project tests/App.Tests -- --treenode-filter "/*/*/*/*[Category=Integration]"
```

The `--` separator passes filter arguments to the TUnit process rather than to
the `dotnet` command. Add `--no-restore` only after restore has succeeded for
the same inputs.

## TUnit filters

`--treenode-filter` matches assembly, namespace, class, and test-name segments.
Use `*` for a wildcard and append property predicates for categories or other
test metadata:

```text
/<assembly>/<namespace>/<class>/<test>[Category=Smoke]
```

Useful examples:

```bash
# A specific test
dotnet run --project tests/App.Tests -- --treenode-filter "/*/*/*/AcceptCookiesTest"

# Namespace prefix
dotnet run --project tests/App.Tests -- --treenode-filter "/*/App.Tests.Api*/*/*"

# Exclude a category
dotnet run --project tests/App.Tests -- --treenode-filter "/*/*/*/*[Category!=Slow]"

# Combine a namespace and property
dotnet run --project tests/App.Tests -- --treenode-filter "/*/App.Tests.Integration/*/*/*[Priority=Critical]"
```

When the user names a group such as integration or smoke, inspect the source
for its `[Category("...")]` annotation and filter that category. Do not run
the full suite when a narrower scope was requested. If no matching annotation
exists, use a class/name filter or state that the requested group is not
encoded in the test suite.

## Execution workflow

1. Translate the requested scope into a project and TUnit tree filter.
2. Check that the project references TUnit and that the requested filter is
   represented by the source metadata.
3. Construct one exact command, explaining only flags relevant to the request.
4. Run it when execution is requested. Preserve the exit code and report the
   first actionable failure instead of hiding it behind a second command.
5. If discovery fails, check the project path, package restore, test attribute,
   and filter path before diagnosing test logic.

## Safe defaults

- Run the narrowest requested project and filter first.
- Never claim tests passed when discovery or the test process failed.
