---
name: stripe-api-design
description: "Design and review public HTTP/JSON APIs using Stripe-inspired resource modeling, predictable operations, idempotent writes, cursor pagination, structured errors, explicit versioning, expandable relationships, and webhook delivery contracts. Use when proposing or auditing an external API, SDK-facing endpoint, webhook/event schema, or compatibility migration; use narrower Rust skills for implementation details. 中文触发：API 设计、接口设计、资源建模、幂等、分页、Webhook、版本化"
---

# Stripe API Design

Use Stripe's API as a source of durable patterns, not as a requirement to copy payment-specific names or behavior.

## Core question

Can a client predict the API's resource model, safely retry uncertain writes, diagnose failures, and evolve independently of the server?

## Design workflow

1. **Model resources first**
   - Name durable nouns and give every persistent object a stable opaque ID and a consistent object discriminator.
   - Use predictable operations: `GET` to retrieve or list, `POST` to create or apply a partial state change, and `DELETE` to remove. Add an action endpoint only when the action is not naturally a resource transition.
   - Use nested paths only when the parent is a real authorization or ownership boundary. Keep a top-level identity for objects that can be referenced elsewhere.
   - Represent relationships by IDs by default. Add an explicit, bounded expansion or field-selection mechanism for clients that need related objects.
   - Define unset versus `null`, server-generated versus client-supplied fields, timestamps, ordering, and metadata before implementation. Do not hide modeled domain data or secrets in free-form metadata.

2. **Specify request semantics**
   - Classify each write as repeatable, idempotent, conditional, or non-repeatable. Make the classification visible in the endpoint contract.
   - Require an idempotency key for retryable creates and side-effecting writes. Bind a key to the original normalized parameters; return the original result for the same request and reject reuse with different parameters. Store the result only after endpoint execution begins, and define the retry behavior for validation and concurrent-key conflicts. Document key scope and retention.
   - Separate transport retries from business conflicts. Use a version, ETag, or domain token when concurrent updates must not silently overwrite one another; return a conflict that tells the client how to recover.
   - Keep query parameters consistent and bounded. Prefer cursor pagination over offset pagination for mutable collections. Define stable ordering, cursor invalidation, page size limits, and whether a page is a snapshot.

3. **Keep responses compact and composable**
   - Use one list envelope everywhere: a list discriminator, `data`, and explicit continuation fields. A Stripe-like v1 contract uses `object: "list"`, `url`, `data`, `has_more`, and mutually exclusive ID cursors such as `starting_after` and `ending_before`; choose names once and keep them consistent. Never require clients to infer continuation from item count.
   - Return IDs for expandable relationships by default. Define a documented path syntax such as `expand[]`, cap expansion depth and response size, and document how expansion works inside list items.
   - Keep response fields additive and semantically stable. Do not change an existing scalar into an object, silently change units, or reuse a field for a new meaning.
   - Document empty, partial, deleted, and asynchronous states. Make success status codes and response bodies consistent across equivalent operations.

4. **Make failures machine-actionable**
   - Use HTTP status for the broad class and a stable JSON error envelope for programmatic handling. Include a stable `type` or `code`, human `message`, invalid `param` or field path when relevant, and a request ID.
   - Distinguish validation, authentication, authorization, not-found, conflict, rate-limit, dependency, and transient server failures. Include `Retry-After` or an equivalent recovery hint when applicable.
   - Never require clients to parse human text. Preserve the request ID end to end so operators can correlate client logs, server logs, and support diagnostics.
   - Define which failures are safe to retry. A timeout does not prove that the server did nothing; use idempotency or status lookup before repeating a side effect.

5. **Version behavior deliberately**
   - Pin an API version for each client, account, or endpoint boundary. Do not let deployment time silently change the contract consumed by an existing client.
   - Treat major behavior changes as migrations with a changelog, testable comparison path, and rollback plan. Treat additive fields and enum values as potentially breaking for strict clients.
   - Version webhook payloads independently or state exactly how their version is selected. Keep synchronous responses and event payloads on an intentional, documented compatibility policy.

6. **Design webhooks as at-least-once delivery**
   - Use an immutable event envelope with a stable event ID, event type, creation time, and the affected resource in `data.object` or an equivalent field.
   - Verify the signature over the exact raw request body with the endpoint secret before parsing. Apply timestamp tolerance and replay protection; never log secrets or full sensitive payloads.
   - Acknowledge quickly with `2xx`, enqueue durable work, and process outside the request. Expect retries, duplicates, and out-of-order delivery; deduplicate by event ID and make the handler's side effects idempotent.
   - Provide a bounded retry policy, observability, and a manual redrive or dead-letter path. Record processing state so a successful response cannot be confused with successful business processing.

## Review output

For a new or changed API, produce these artifacts before approving it:

- resource and relationship map;
- endpoint table with method, path, auth scope, request, response, and status codes;
- retry and concurrency contract for every write;
- list, expansion, error, and request-ID conventions;
- version and compatibility impact, including enum and webhook changes;
- webhook delivery, deduplication, redrive, and signature-verification behavior;
- contract tests for success, validation, timeout-after-commit, duplicate delivery, stale update, pagination boundary, and version mismatch cases.

Reject proposals that rely on implicit retries, offset pagination for mutable data, unbounded embedding, message-string parsing, unversioned response changes, or webhook ordering assumptions.

## Boundary with related skills

- Use this skill for the external API contract and wire behavior.
- Use `rust-method-placement`, `rust-api-consolidation`, or `rust-structure-refactor` for Rust method and module boundaries, not for HTTP resource semantics.
- Use `rust-state-machine` for internal legal states and transitions; use `error-silence` or `rust-snafu` for Rust error implementation.
- Use `async-concurrency` and `concurrency-testing` for runtime task ownership, cancellation, synchronization, and forced interleavings behind the contract.
- Use `testing-strategy` for broader test-level and fixture choices; use `architecture-entropy-review` when multiple modules or routes create competing API sources of truth.

## Official sources

- [Stripe API reference](https://docs.stripe.com/api)
- [Idempotent requests](https://docs.stripe.com/api/idempotent_requests)
- [Pagination](https://docs.stripe.com/api/pagination)
- [Expanding responses](https://docs.stripe.com/api/expanding_objects)
- [Errors](https://docs.stripe.com/api/errors)
- [Request IDs](https://docs.stripe.com/api/request_ids)
- [Versioning](https://docs.stripe.com/api/versioning)
- [Webhook delivery](https://docs.stripe.com/webhooks)
- [Webhook signature verification](https://docs.stripe.com/webhooks/signature)
