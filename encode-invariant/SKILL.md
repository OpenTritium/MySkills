---
name: encode-invariant
description: Use in Rust when data stops changing after initialization — pick types that encode immutability structurally. Use Box<[T]> over Vec<T>, Box<str> over String, Arc<str> for shared strings, NonZero/nonzero types for domain constraints, newtypes for wrapping, and enums to make illegal states unrepresentable. Keywords: Rust, Box<[T]>, into_boxed_slice, Box<str>, into_boxed_str, invariant, immutable, Vec to boxed slice, String to boxed str, newtype, NonZero, make invalid states unrepresentable, type state, CompactString, smallvec, niche optimization, 类型不变, 盒装切片, 新类型
---

# Encode Invariant

## Overview
Rust turns immutability and domain constraints into compile-time facts. If data won't change after construction, the type should say so. Every `Vec<T>` that never grows is a leaked capability — a maintainer can `.push()` because the type allows it. Shrink the type, not the runtime check.

## When to Use
- `Vec<T>` / `String` populated once, never modified after
- `Arc<Vec<T>>` / `Rc<String>` that could be `Arc<[T]>` / `Arc<str>`
- `u64` / `String` in domain code with no type distinction
- `Option<T> + bool` pairs encoding impossible states
- Comments saying "must not be empty / zero / negative" — should be a type

## Rules Engine

1. **Freeze with Box** — `Vec<T>` → `Box<[T]>`, `String` → `Box<str>` when built once then read-only. `.into_boxed_slice()` / `.into_boxed_str()` at the finish. Saves 8 bytes (no capacity field); `.push()` becomes a compile error.

2. **Share without Fat** — `Arc<Vec<T>>` = 3 layers (Arc → Vec → heap); `Arc<[T]>` = 2. Same for `Rc<String>` → `Rc<str>`. Convert at construction: `Arc::from(vec.into_boxed_slice())`.

3. **Newtype the Meaning, Not the Mechanics** — `struct UserId(u64)` prevents `send(userID, orderID)` arg swaps; zero-cost. **But only newtype when you need domain behavior** — methods, validation, distinct signature identity, or arg-swap protection. For pure readability with no methods, `type UserId = u64;` is simpler (no protection). Newtype buys enforcement; alias buys ergonomics.

4. **Prove Constraints at the Boundary** — "Amount > 0" → `NonZeroU64` or `struct PositiveAmount(u64)` with validating constructor. "Name non-empty" → `struct NonEmptyString(String)`. Validate once at construction; consumers are safe by construction.

5. **Enum > (bool + Option)** — If `is_connected` implies `connection.is_some()`, you have impossible states. Replace:
   ```rust
   enum Connection { Disconnected, Connected(Socket), Authenticated { token: Token, socket: Socket } }
   ```
   Now `send(Disconnected)` won't compile.

6. **Niche Matters** — `Option<Box<T>>` = same size as `Box<T>` (null is niche). `Option<NonZeroU32>` = same size as `u32` (zero is niche). For optional values with an absent sentinel, `Option` is free.

## Transformation Table

| From | To | Savings |
|---|---|---|
| `Vec<T>` (read-only after build) | `Box<[T]>` | 8 bytes, `.push()` compile error |
| `String` (read-only after build) | `Box<str>` / `Arc<str>` | 8 bytes, `.push_str()` compile error |
| `Arc<Vec<T>>` | `Arc<[T]>` | 1 indirection + 8 bytes |
| `Rc<String>` | `Rc<str>` / `Arc<str>` | 1 indirection + 8 bytes |
| `u64` args (swappable, needs methods) | `OrderId(u64)`, `UserId(u64)` newtypes | arg-swap prevention + method home |
| `u64` (readability only, no methods) | `type UserId = u64;` alias | ergonomic; no type protection |
| `amount: u64 // must be > 0` | `NonZeroU64` / `PositiveAmount(u64)` | compile-time guarantee |
| `Option<T>` + `bool` | `enum { Connected(T), Disconnected }` | 1 impossible state eliminated |
| `PathBuf` (read-only) | `Box<Path>` | 8 bytes, no mutation |
| `Vec<u8>` (read-only, e.g. hash digest) | `Box<[u8]>` / `[u8; N]` | 8+ bytes, immutability |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `let ids: Vec<u64> = load(); // never mutates` | `let ids: Box<[u64]> = load().into_boxed_slice();` |
| `fn cache(data: Arc<Vec<User>>)` | `fn cache(data: Arc<[User]>)` — `Arc::from(vec.into_boxed_slice())` |
| `fn delete(user_id: u64, org_id: u64)` | Newtype both — prevents swap bug |
| `struct UserId(u64)` zero methods, used only in formatting | Downgrade to `type UserId = u64;` — newtype adds friction, no payoff |
| `struct Config { name: String } // never changed` | `name: Box<str>` — trims capacity field |
| `if amount <= 0 { return Err }` in 5 places | Validate once in `PositiveAmount::new()` |
| `fn process(conn: Option<Conn>, ready: bool)` | `enum ConnState { Disconnected, Connecting, Ready(Conn) }` |
| Comment: `// append-only after init` | Let the type enforce it — comment is not a compiler |

## Workflow
1. Find `Vec<T>` / `String` / `PathBuf` / `OsString` never mutated after init → boxed type
2. Find `Arc<Vec<T>>` / `Rc<String>` → flatten to `Arc<[T]>` / `Arc<str>`
3. Comments describing invariants ("must not be empty", "always positive") → newtype
4. `(Option<T>, bool)` pairs → single enum
5. Swappable `u64`/`String` params → newtype
6. Output diff: old type → new type, bytes saved, bug class prevented
