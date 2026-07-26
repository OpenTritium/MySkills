---
name: dotnet-webapi
description: "Create or review .NET 10 ASP.NET Core Web API endpoints, DTOs, OpenAPI metadata, HTTP status codes, validation, and centralized error handling. Use for controllers or minimal APIs; use stripe-api-design for the language-neutral public wire contract. Do not use for general C# or test code."
license: MIT
---

# ASP.NET Core Web API

Use this skill to implement or review the ASP.NET Core 10 layer of a .NET 10
HTTP API. The skill is a curated adaptation of the Web API guidance in
`dotnet/skills` and focuses on decisions that affect correctness,
compatibility, and testability.

## Core question

Does this endpoint expose a stable, documented HTTP contract and handle
validation, cancellation, and failures consistently with the rest of the API?

## When to use

- Add or change controller or minimal API endpoints.
- Define request/response DTOs, validation, status codes, or OpenAPI metadata.
- Add centralized exception handling or `.http` request examples.
- Review an ASP.NET Core API for contract and implementation mistakes.

## Boundary

- Use `stripe-api-design` for resource modeling, idempotency, pagination,
  versioning, and webhook semantics independent of ASP.NET Core.
- Use this skill for the C# endpoint, middleware, DTO, and OpenAPI
  implementation of that contract.
- Keep persistence and query design outside this endpoint skill.
- Do not use this skill for general C# or test code.

## Workflow

1. Read `Program.cs`, existing endpoint registrations, controllers, DTOs,
   validation types, and error handling before choosing a pattern.
2. Keep the existing endpoint style. For a new API with no established style,
   prefer minimal APIs unless controllers are explicitly required.
3. Separate transport DTOs from persistence entities. Prefer `sealed record`
   DTOs, `DateTimeOffset` for timestamps, and explicit validation constraints.
4. Give each endpoint a stable name, summary, description, response types, and
   documented error responses. Add OpenAPI support using .NET 10 package
   choices.
5. Accept and forward `CancellationToken` through every asynchronous boundary.
6. Use a service boundary for non-trivial data access and map entities to DTOs
   there; do not expose navigation properties or internal fields by accident.
7. Verify the contract with build/tests and a `.http` example for each new
   route, including at least one validation or not-found case.

## HTTP behavior

| Operation | Success | Common errors |
|---|---|---|
| Get one | `200 OK` | `404 Not Found` |
| List | `200 OK` | validation errors as applicable |
| Create | `201 Created` plus `Location` | `400`, `409` |
| Replace/update | `200 OK` | `400`, `404`, `409` |
| Delete | `204 No Content` | `404`, `409` |

For minimal APIs, prefer `TypedResults` and annotate handlers returning more
than one result with `Results<T1, T2>`. Use the `Results` factory only when the
number of branches makes the typed signature less readable. For controllers,
return the corresponding `ActionResult<T>` and use `CreatedAtAction` for a
successful create.

## DTO and serialization rules

- Do not use persistence entities as public request or response types.
- Use immutable response records and `init`-only request properties.
- Preserve existing JSON behavior in an established API. Stricter settings
  such as case-sensitive input or duplicate-property rejection are for new
  APIs or an explicitly requested contract change only.
- For new .NET 10 APIs, prefer the built-in OpenAPI registration with
  `AddOpenApi()` and `MapOpenApi()`. Preserve an established API's existing
  OpenAPI package configuration unless the user explicitly requests a change.

## Errors and validation

Configure one global error path with `ProblemDetails` and the built-in exception
handler or an `IExceptionHandler`. Map known domain failures to stable status
codes and safe messages. Do not return exception messages, stack traces, or
secrets to clients. Log handled exceptions before suppressing middleware
diagnostics.

Validation must happen before the handler performs side effects. Controllers
get data-annotation validation from MVC; minimal APIs need explicit validation
registration. Keep validation errors in the same problem-details shape as other
API errors.

## Review checklist

- [ ] Existing controller/minimal-API style is preserved.
- [ ] Request/response DTOs are separate from persistence entities.
- [ ] Status codes, `Location`, and cancellation behavior are explicit.
- [ ] OpenAPI metadata and ProblemDetails cover success and error paths.
- [ ] New routes have executable or `.http` request coverage.
