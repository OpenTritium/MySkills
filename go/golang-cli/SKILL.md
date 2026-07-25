---
name: golang-cli
description: "Build, extend, or review Go command-line applications, including command structure, flags, configuration, exit codes, I/O, signals, completion, and CLI testing. Use the standard library for small tools and preserve an existing CLI framework when the repository already depends on one; do not assume a specific library."
---

# Go CLI Applications

Design the CLI as a stable boundary for humans and scripts. Keep parsing and presentation at the edge, pass typed options into application code, and return errors to one process-level exit boundary.

## Choose the Tooling

- Use `flag` for a small command with a flat set of options and no framework requirement.
- Preserve the repository's existing command framework when extending a mature CLI. Follow its local conventions instead of introducing a second parser.
- Add a framework only when the command tree, completion, generated help, or middleware needs justify the dependency.
- Keep configuration loading independent from command registration. Use a typed configuration struct and make precedence explicit, for example flags over environment variables over files over defaults.

## Structure

For a multi-command application, keep the entry point small and isolate command wiring from business logic:

```text
cmd/myapp/main.go       # process boundary and exit status
internal/cli/            # command parsing, help, and output
internal/app/            # application use cases
internal/config/         # typed configuration and validation
```

The command layer should parse arguments, validate user input, construct dependencies, call an application function, and format the result. It should not contain database or domain workflows.

## Process Boundary

- Write data output to stdout and diagnostics, logs, and errors to stderr.
- Return errors from command handlers; call `os.Exit` only in `main` after cleanup-sensitive work has finished.
- Use stable exit codes. Reserve `0` for success, distinguish usage errors from runtime failures, and document non-zero codes used by scripts.
- Keep `--help` and `--version` cheap and usable without network or database access.
- Prefer machine-readable output modes such as JSON when users may pipe results to another program.

## Configuration and Validation

1. Parse flags and collect environment or file values into a typed struct.
2. Apply the documented precedence order once; avoid hidden global configuration state.
3. Validate cross-field constraints before constructing clients or opening resources.
4. Redact secrets in errors and diagnostics. Do not put credentials in help text, generated examples, or command history when avoidable.
5. Make optional configuration files explicit. A missing optional file should not become a startup panic.

Use positional-argument validation that reports the expected shape. Reject unknown or ambiguous input early, but preserve useful command-specific help.

## Signals and Cancellation

Use `signal.NotifyContext` at the process boundary and pass the derived context through application calls. On cancellation, stop accepting work, let active operations finish within a bounded deadline, close resources, and return a meaningful status.

## Completion and Output

Completion should be deterministic, fast, and side-effect free. Do not query production services merely to complete a flag unless the command explicitly opts into that behavior.

For output:

- Keep human-readable output stable enough for routine use.
- Keep logs off stdout so pipelines remain valid.
- Use an encoder for JSON instead of concatenating strings.
- Write through an injected `io.Writer` so tests can capture output.

## Testing

- Test parsing, defaults, precedence, validation, exit classification, and output separately from application behavior.
- Construct a fresh command/configuration instance per test when the parser stores mutable state.
- Exercise the process boundary with subprocess tests only for signals, actual exit codes, or terminal behavior; keep most tests in-process.
- Cover representative success, usage error, runtime error, cancellation, and machine-readable output paths.

## Common Mistakes

| Mistake | Better practice |
| --- | --- |
| Mixing command parsing with domain logic | Keep handlers thin and call typed application functions |
| Printing logs to stdout | Reserve stdout for command results and use stderr for diagnostics |
| Calling `os.Exit` inside a handler | Return an error and classify it at the process boundary |
| Reading environment variables throughout the codebase | Load configuration once into a validated struct |
| Making help depend on external services | Keep help and version paths local and fast |
| Starting a new framework for a tiny command | Use `flag` unless the extra dependency solves a real problem |
| Letting cancellation stop only the top-level loop | Propagate context to every blocking operation and resource owner |

## Related Skills

See `golang-project-layout` for repository structure, `golang-error-handling` for error contracts, `golang-context` for cancellation, `golang-security` for secret handling, and `golang-testing` for test strategy.
