# do/v2 Reference

`github.com/samber/do/v2` is a generic Go dependency-injection toolkit. Keep this reference focused on the container API; use the main skill for deciding whether a container is justified.

## Core API

```go
injector := do.New()

do.Provide(injector, NewConfig)
do.Provide(injector, NewDatabase)

db, err := do.Invoke[*Database](injector)
service := do.MustInvoke[*Service](injector)
```

Use `Invoke` when startup errors must be handled. Use `MustInvoke` only where a failure is intentionally fatal and the caller is already at the process boundary.

## Names and Interfaces

Use names for multiple implementations of the same type:

```go
do.ProvideNamed(injector, "primary", NewPrimaryStore)
store, err := do.InvokeNamed[*Store](injector, "primary")
```

Keep interfaces at the consuming package. Bind a concrete implementation to an interface only when the boundary requires it, and document which implementation owns the name or alias.

## Scopes and Overrides

Use child scopes to group module registrations and control visibility. Use overrides in tests to replace external boundaries without rebuilding unrelated production constructors. Do not expose the injector beyond the composition root.

## Lifecycle

Register services that implement the library's lifecycle contracts when the application owns their health checks or shutdown. Make shutdown idempotent and verify cleanup in integration tests.

## Performance Boundary

The container's cost is paid during registration and resolution. Do not put `Invoke` in request handlers, message loops, or other hot paths. Once resolved, the service should be called through its ordinary Go interface or concrete type.
