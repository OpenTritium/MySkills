---
name: golang-openapi
description: "Design, generate, review, and evolve OpenAPI 3.x contracts and type-safe Go clients or servers with ogen. Use when a Go project imports github.com/ogen-go/ogen, maintains an OpenAPI specification, or needs contract-first HTTP API generation, validation, compatibility checks, or generated client/server integration. Use golang-grpc for protobuf/gRPC and golang-graphql for GraphQL instead."
---

# Go OpenAPI with ogen

Use the OpenAPI document as the API contract and `ogen` as the generated-code boundary. Keep business logic in handwritten adapters and never edit generated files directly.

## Decide the Boundary

- Use this skill for OpenAPI 3.x schemas, `ogen` generation, generated clients or servers, HTTP contract validation, and compatibility review.
- Use `stripe-api-design` for resource semantics, idempotency, pagination, and public API behavior before encoding those decisions in OpenAPI.
- Use `golang-security` for threat modeling, authentication, authorization, and sensitive-data review.
- Use `golang-testing` for test strategy; use generated clients and `httptest` for API conformance tests.

## Workflow

1. Read the existing specification, generated package, `go.mod`, and `go:generate` directives before changing anything. Determine whether the project is contract-first or is adopting OpenAPI around an existing HTTP service.
2. Pin the `ogen` version in the repository. Prefer a reproducible directive such as:

   ```go
   //go:generate go run github.com/ogen-go/ogen/cmd/ogen@vX.Y.Z --target internal/api --package api --clean ../../api/openapi.yaml
   ```

   Match the actual `ogen` release and repository layout; do not use an unpinned `@latest` command in CI.
3. Design the specification before generating code. Give every operation a stable, unique `operationId`; define request and response schemas; model errors; specify authentication; and use explicit formats, required fields, nullable values, pagination, and idempotency behavior.
4. Generate into a dedicated package such as `internal/api` or `internal/gen`. Review the generated diff, run `gofmt`, and compile immediately. Keep generated output deterministic and committed or reproducibly generated according to the repository policy.
5. Implement the generated server interface in a thin adapter. Keep domain validation, authorization, transactions, and orchestration in application packages. Map domain errors to the documented response types and status codes instead of leaking implementation errors.
6. Use the generated client for typed calls. Keep authentication, transport timeouts, retry limits, backoff, and observability explicit at the client boundary; retries must be safe for the operation's idempotency semantics.
7. Validate every contract change in CI: parse and lint the OpenAPI document, regenerate code, run `go test ./...`, and compare the new contract with the previous published version for breaking changes. Treat removed operations, tightened constraints, incompatible response shapes, and security changes as intentional review items.

## Contract Rules

- Prefer one canonical schema per resource; avoid duplicating equivalent request and response models without a documented reason.
- Keep `operationId` stable. It affects generated Go names and downstream client compatibility.
- Distinguish absent, `null`, empty, and zero values deliberately. Check how the chosen `ogen` version represents optional and nullable fields before writing handlers.
- Define error responses consistently, including validation and authentication failures. Do not return undocumented error shapes.
- Keep examples valid against the schema and make security requirements explicit per operation.
- Treat the generated package as an API boundary. Wrap it with handwritten code when the generated shape is inconvenient; do not patch generated files.

## Common Mistakes

| Mistake | Better practice |
| --- | --- |
| Editing generated Go files | Change the OpenAPI document or generator configuration, then regenerate |
| Running `ogen@latest` in CI | Pin a version and update it deliberately |
| Missing or unstable `operationId` values | Assign unique, human-readable IDs and preserve them after publication |
| Confusing omitted, `null`, empty, and zero values | Model each state explicitly and test the generated optional/nullable type |
| Returning arbitrary errors from handlers | Map failures to documented status codes and error schemas |
| Regenerating without reviewing the diff | Run formatting, compilation, contract tests, and compatibility checks together |
| Adding retries to every request | Retry only bounded, transient, and semantically safe operations |

## Minimal Verification

```bash
go generate ./...
gofmt -w internal/api
go test ./...
```

Use the repository's OpenAPI linter and compatibility checker when present. If no tool is configured, at minimum parse the document, regenerate from a clean tree, compile generated code, and exercise representative success, validation-error, auth-error, and not-found paths.
