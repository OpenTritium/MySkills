---
name: encode-invariant
description: Use in Rust when data stops changing after initialization — pick types that encode immutability structurally. Use Box<[T]> over Vec<T>, Box<str> over String, Arc<str> for shared strings, NonZero/nonzero types for domain constraints, newtypes for wrapping, and enums to make illegal states unrepresentable. Keywords: Rust, Box<[T]>, into_boxed_slice, Box<str>, into_boxed_str, invariant, immutable, Vec to boxed slice, String to boxed str, newtype, NonZero, make invalid states unrepresentable, type state, CompactString, smallvec, niche optimization, 类型不变, 盒装切片, 新类型
---

# Encode Invariant

## Overview
Rust's type system lets immutability and domain constraints become compile-time facts, not runtime checks. If data won't change after construction, the type should say so. Every `Vec<T>` that never grows is a leaked capability — a future maintainer can `.push()` into it because the type allows it. Shrink the type, not the runtime check. A smaller type is smaller memory, clearer intent, and one less invariant to document in a comment.

## When to Use
- `Vec<T>` or `String` that is populated once and never modified afterward
- `Arc<Vec<T>>` or `Rc<String>` that could be `Arc<[T]>` / `Arc<str>`
- Wrapping `u64` / `String` in domain code with no type distinction
- `Option<T> + bool` pairs that encode impossible state combinations
- Any comment that says "this must not be empty / zero / negative" — that comment should be a type

## Rules Engine

1. **Freeze with Box** — Replace `Vec<T>` → `Box<[T]>` and `String` → `Box<str>` when the collection is built once then read-only. Call `.into_boxed_slice()` or `.into_boxed_str()` at the finish line. Saves 8 bytes per value (no capacity field). Then `.push()` is a compile error.

2. **Share without Fat** — `Arc<Vec<T>>` has three layers (Arc → Vec → heap data). `Arc<[T]>` has two (Arc → heap data). Same for `Rc<String>` → `Rc<str>`. Convert at construction: `Arc::from(vec.into_boxed_slice())`.

3. **Newtype the Meaning, Not the Mechanics** — `struct UserId(u64)` beats `userID u64` everywhere. The type system now prevents `sendNotification(userID, orderID)` where args are swapped. Wrapper is zero-cost; bug is expensive. **But only newtype when you need domain behavior** — methods, validation, a distinct identity in signatures, or compile-time arg-swap protection. If it's purely for readability and you never attach methods, a type alias (`type UserId = u64;`) is simpler and cheaper to use, at the cost of zero type protection. Newtype buys enforcement; alias buys ergonomics. Pick the level the bug risk justifies.

4. **Prove Constraints at the Type Boundary** — "Amount must be positive" → `NonZeroU64` or `struct PositiveAmount(u64)` with a constructor that validates. "Name must not be empty" → `struct NonEmptyString(String)`. The validation happens once at construction; all consumers are safe by construction.

5. **Enum > (bool + Option)** — If `is_connected` implies `connection.is_some()`, you have impossible states. Replace with:
   ```rust
   enum Connection { Disconnected, Connected(Socket), Authenticated { token: Token, socket: Socket } }
   ```
   The compiler now rejects `send_packet(Connection::Disconnected)`.

6. **Niche Matters** — `Option<Box<T>>` is the same size as `Box<T>` (null pointer is niche). `Option<NonZeroU32>` is the same size as `u32` (zero is niche). For optional values with a provably absent sentinel, the `Option` wrapper is literally free.

## Transformation Table

| From | To | Savings |
|---|---|---|
| `Vec<T>` (read-only after build) | `Box<[T]>` | 8 bytes, `.push()` compile error |
| `String` (read-only after build) | `Box<str>` / `Arc<str>` | 8 bytes, `.push_str()` compile error |
| `Arc<Vec<T>>` | `Arc<[T]>` | 1 indirection removed + 8 bytes |
| `Rc<String>` | `Rc<str>` / `Arc<str>` | 1 indirection removed + 8 bytes |
| `u64 orderID, u64 userID` (args swappable, needs methods) | `OrderId(u64)`, `UserId(u64)` newtypes | Zero-cost arg-swap prevention + method home |
| `u64 userID` (readability only, no methods/validation) | `type UserId = u64;` alias | Ergonomic; no type protection — use only when the bug risk is low |
| `amount: u64 // must be > 0` | `NonZeroU64` or `PositiveAmount(u64)` | Compile-time guarantee |
| `conn: Option<T>, is_connected: bool` | `enum { Connected(T), Disconnected }` | 1 impossible state eliminated |
| `PathBuf` (read-only) | `Box<Path>` | 8 bytes, no mutation methods |
| `Vec<u8>` (read-only, e.g. hash digest) | `Box<[u8]>` or `[u8; N]` if fixed | 8+ bytes, immutability guarantee |

## Common Mistakes

| Anti-Pattern | Fix |
|---|---|
| `let ids: Vec<u64> = load(); ... // never mutates` | `let ids: Box<[u64]> = load().into_boxed_slice();` |
| `fn cache(data: Arc<Vec<User>>)` | `fn cache(data: Arc<[User]>)` — `Arc::from(vec.into_boxed_slice())` |
| `fn delete(user_id: u64, org_id: u64)` | Newtype both — prevents `delete(orgID, userID)` swap bug (here arg-swap risk justifies the newtype) |
| `struct UserId(u64)` with zero methods, only used in formatting | Downgrade to `type UserId = u64;` — newtype adds friction with no payoff |
| `struct Config { name: String } // never changed` | `struct Config { name: Box<str> }` — trims capacity field |
| `if amount <= 0 { return Err }` scattered in 5 places | Validate once in `PositiveAmount::new()`, use the type everywhere |
| `fn process(conn: Option<Conn>, ready: bool)` | `enum ConnState { Disconnected, Connecting, Ready(Conn) }` |
| Comment: `// this Vec is append-only after init` | Let the type enforce it — comment is not a compiler |

## Workflow
1. Find every `Vec<T>` / `String` / `PathBuf` / `OsString` that is never mutated after initialization — convert to boxed type
2. Find `Arc<Vec<T>>` / `Rc<String>` — flatten to `Arc<[T]>` / `Arc<str>`
3. Scan for comments describing type invariants ("must not be empty", "always positive") — promote each to a newtype
4. Spot `(Option<T>, bool)` or `(Option<T>, enum{...})` pairs — unify into a single enum
5. Audit function signatures: can two `u64`/`String` params be swapped? — newtype them
6. Output the diff: old type → new type, bytes saved, and which class of bugs each change prevents
