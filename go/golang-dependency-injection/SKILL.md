---
name: golang-dependency-injection
description: "Design Go dependency injection with manual constructors or github.com/samber/do/v2. Use when wiring a growing Go service, choosing between explicit construction and a DI container, configuring do/v2 providers, scopes, lifecycle, overrides, or testing an injected graph. Prefer manual injection for small graphs; use do/v2 when type-safe generic registration, lazy resolution, scopes, and lifecycle management justify a container."
---

# Dependency Injection in Go

Keep dependency injection at the composition root. Components should receive the interfaces or concrete values they need; they should not discover services through globals, `init()`, or a container passed through the application.

## Choose the Boundary

- Use manual constructor injection for small or stable graphs. It has the lowest runtime cost, the clearest call sites, and no dependency.
- Use `github.com/samber/do/v2` when the graph is large enough that explicit wiring is repetitive and the application needs generic type-safe registration, lazy resolution, scopes, overrides, or lifecycle hooks.
- Do not resolve dependencies on request or hot paths. Build and validate the graph at startup, then pass ordinary typed dependencies to handlers and services.
- Do not use a container as a service locator. A component should not accept `do.Injector` merely to look up arbitrary services.

## Manual Constructor Injection

For a small graph, keep the composition root explicit:

```go
func main() {
    cfg := NewConfig()
    db := NewDatabase(cfg)
    repo := NewUserRepository(db)
    service := NewUserService(repo)
    server := NewServer(service)

    if err := server.Run(context.Background()); err != nil {
        log.Fatal(err)
    }
}
```

See [manual DI examples](./references/manual-di.md) for constructor and test patterns.

## do/v2 Workflow

Install the current major version explicitly:

```bash
go get github.com/samber/do/v2@v2
```

Register constructors in one composition-root module and resolve the application entrypoint once:

```go
package main

import (
    "context"

    "github.com/samber/do/v2"
)

func buildInjector() do.Injector {
    injector := do.New()

    do.Provide(injector, NewConfig)
    do.Provide(injector, NewDatabase)
    do.Provide(injector, NewUserRepository)
    do.Provide(injector, NewUserService)
    do.Provide(injector, NewServer)

    return injector
}

func main() {
    injector := buildInjector()
    defer injector.Shutdown()

    server := do.MustInvoke[*Server](injector)
    if err := server.Run(context.Background()); err != nil {
        panic(err)
    }
}
```

Provider functions should accept `do.Injector` only when they need to resolve another registered dependency. Prefer typed constructor parameters when the graph is simple and use `do.MustInvoke[T]` only at the composition root or inside provider functions.

## Registration Rules

- Register one constructor per service and keep providers side-effect free until lifecycle startup.
- Return `(T, error)` from fallible providers; never hide initialization errors in a panic or zero value.
- Use `do.ProvideNamed` only when multiple values share a type and the name is part of the contract.
- Use `do.As` for an intentional concrete-to-interface binding; define the interface where it is consumed.
- Use scopes to express module visibility, not to create an ad hoc service locator.
- Keep configuration, database pools, clients, and servers as application-lifetime services. Use transient providers only when each resolution must create a fresh value.

## Lifecycle

Use do/v2 lifecycle support for resources that need health checks or ordered shutdown. Register cleanup with the resource owner and make shutdown idempotent. Propagate a bounded context to network and database cleanup instead of blocking indefinitely.

Start long-running services explicitly after the graph is built. Do not start goroutines from package-level variables or constructors unless startup ownership and shutdown behavior are documented.

## Testing

Prefer unit tests with manual constructor injection. For integration tests that exercise a real graph:

- create a fresh injector per test;
- override only external boundaries such as databases, clocks, queues, or HTTP clients;
- resolve the tested entrypoint once;
- call `Shutdown()` in cleanup;
- assert startup and teardown errors instead of ignoring them.

Keep tests independent from the default/global injector. A test must not inherit registrations or mutable singleton state from another test.

## Common Mistakes

| Mistake | Better practice |
| --- | --- |
| Using DI for a tiny graph | Keep explicit constructors and manual wiring |
| Passing `do.Injector` into business code | Inject the specific interface or value required |
| Calling `Invoke` for every request | Resolve application-lifetime services at startup |
| Hiding errors in `MustInvoke` | Use explicit error handling in startup paths where failure must be reported |
| Starting resources inside package `init()` | Register ownership at the composition root and close it during shutdown |
| Reusing one injector across tests | Build a fresh injector and override test boundaries |
| Registering ambiguous same-type services | Use explicit names or interfaces with documented ownership |

## Related Skills

See `golang-structs-interfaces` for interface boundaries, `golang-context` for cancellation and lifecycle deadlines, `golang-testing` for test design, `golang-project-layout` for composition-root placement, and `golang-design-patterns` for constructor and options design.
