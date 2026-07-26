---
name: dotnet-aot-compat
description: "Make .NET 10 projects compatible with trimming and Native AOT by resolving IL2026, IL2067, IL2070, IL2072, IL3050, and related analyzer warnings with annotations or source-generated alternatives. Do not use for binary-size optimization or unrelated publishing work."
license: MIT
---

# .NET Trimming and Native AOT Compatibility

Use this skill when a project needs to pass trim/AOT analysis. Native AOT and
the trimmer need statically discoverable code; reflection, dynamic code, and
unannotated type flow can otherwise fail at publish time even when a normal
build succeeds.

## Core question

Can the trimmer and AOT compiler prove every required member and code path is
available, or is the incompatible behavior explicitly isolated at a supported
boundary?

## When to use

- Enable AOT analysis for a `net10.0` project.
- Resolve IL2026, IL2057, IL2067, IL2070, IL2072, IL2091, IL3000, IL3050, or
  related warnings.
- Annotate reflection APIs or replace runtime serialization with source
  generation.

Do not use this skill for publishing instructions by themselves or for
suppressing warnings to make a build appear clean.

## Warning-driven workflow

1. Confirm that the project targets `net10.0`.
2. Enable analysis and build the project with a clean incremental state:

   ```xml
   <PropertyGroup>
     <IsAotCompatible>true</IsAotCompatible>
   </PropertyGroup>
   ```

   ```bash
   dotnet build path/to/project.csproj -f net10.0 --no-incremental
   ```

3. Group warnings by code and fix the innermost reported call first. Rebuild
   after a small batch; annotations intentionally propagate requirements to
   callers.
4. Verify with a trim publish and, when requested and supported by the project,
   a Native AOT publish. A clean normal build is not proof of AOT compatibility.

## Preferred fixes

| Signal | Preferred response |
|---|---|
| `Type` reflection loses member information | Add `DynamicallyAccessedMembers` at the source and preserve the annotated type flow. |
| Method always requires reflection/dynamic code | Mark it with `RequiresUnreferencedCode` or `RequiresDynamicCode` and make callers acknowledge the boundary. |
| `JsonSerializer` triggers IL2026/IL3050 | Use a `JsonSerializerContext` and source-generated `JsonTypeInfo` for the model. |
| Dynamic type names or assembly probing | Replace with a closed registry, generated metadata, or an explicit supported set. |
| A library cannot be made compatible | Isolate and document the unsupported API; do not claim the whole project is AOT-compatible. |

`DynamicallyAccessedMembers` describes the members that must be preserved; it
is not a general escape hatch. Annotate the narrowest parameter, field,
property, return value, or generic argument that carries the `Type` value. Keep
the annotation through calls, assignments, and returns instead of boxing the
value into `object` or an untyped collection.

## Suppression rule

Do not use `#pragma warning disable`, `NoWarn`, or
`UnconditionalSuppressMessage` as the default fix for IL warnings. A
suppression can hide a real linker failure and makes the compatibility claim
unverifiable. If a suppression is unavoidable, document the proven runtime
reachability argument, scope it to the smallest call, and add a publish test.

## Validation

- [ ] The `net10.0` project has `IsAotCompatible` enabled.
- [ ] All analyzer warnings are fixed or explicitly justified at a narrow API
  boundary.
- [ ] Reflection and JSON metadata are statically discoverable where possible.
- [ ] Trim publish succeeds; Native AOT publish is tested when in scope.
- [ ] The result does not rely on a blanket warning suppression.
