# API-Facing Attributes

Attributes communicate contracts to callers and tooling.

## `#[must_use]`

Use on builders, guards, futures, and return values where ignoring the value is likely a bug.

```rust
#[must_use = "call `.send()` to execute the request"]
pub struct RequestBuilder { /* fields */ }

#[must_use = "guard must be held for the duration of the scope"]
pub struct MutexGuard<'a, T>(&'a mut T);

#[must_use = "future does nothing unless awaited"]
pub async fn compute() -> Result<(), Error> { /* ... */ }
```

Apply to functions returning `Result` to force callers to handle errors. The compiler will warn on dropped results.

## `#[non_exhaustive]`

Use for public enums or structs that may grow. Prevents downstream match exhaustiveness errors when you add variants.

```rust
#[non_exhaustive]
pub enum PaymentMethod {
    CreditCard,
    PayPal,
    Crypto,
}

#[non_exhaustive]
pub struct Config {
    pub timeout_ms: u64,
    pub retry_count: u32,
    // future fields won't break callers
}
```

Pattern matching a `#[non_exhaustive]` enum requires a wildcard arm:

```rust
match method {
    PaymentMethod::CreditCard => /* ... */,
    PaymentMethod::PayPal => /* ... */,
    PaymentMethod::Crypto => /* ... */,
    _ => Err(Error::UnsupportedPaymentMethod), // required for forward compat
}
```

## `#[deprecated]`

Deprecate before removing. Always include a replacement hint.

```rust
#[deprecated(since = "2.1.0", note = "use `Client::builder()` instead")]
pub fn new_client() -> Client { Client::builder().build() }

#[deprecated(since = "3.0.0", note = "use `send_batch(&[Event])` for better throughput")]
pub fn send_event(event: Event) -> Result<(), Error> { /* ... */ }
```

Use `#[allow(deprecated)]` inside the implementation of the replacement to avoid warnings on self-use during migration.

## `#[doc(cfg(...))]`

Show feature-gated APIs in docs.

```rust
#[cfg(feature = "serde")]
#[cfg_attr(docsrs, doc(cfg(feature = "serde")))]
impl serde::Serialize for UserId { /* ... */ }
```

Add to `Cargo.toml` to enable feature badges in docs:

```toml
[package.metadata.docs.rs]
features = ["serde", "full"]
all-features = true
rustdoc-args = ["--cfg", "docsrs"]
```

## `#[inline]`

```rust
#[inline]
pub fn small_operation(x: u32) -> u32 {
    x.wrapping_add(1)
}

#[inline(always)]
pub fn fast_path(x: u32) -> u32 {
    x * 2
}

#[inline(never)]
pub fn cold_path(err: Error) -> ! {
    panic!("unexpected: {}", err);
}
```

Use `#[inline]` for small functions called across crate boundaries. Use `#[inline(never)]` for cold or large functions that should not bloat hot code.

## `#[repr(...)]`

Only promise layout when needed. Standard Rust types have no guaranteed layout.

```rust
// Same size as the inner type, no extra padding
#[repr(transparent)]
pub struct UserId(u64);

// C-compatible layout for FFI
#[repr(C)]
pub struct CHeader {
    len: u32,
    flags: u32,
}

// Packed struct for wire formats
#[repr(C, packed)]
pub struct Packet {
    version: u8,
    kind: u8,
    payload: [u8; 64],
}

// Align to cache line to avoid false sharing
#[repr(align(64))]
pub struct CachePadded<T>(pub T);
```

Avoid `repr(C)` for normal Rust-only types; it freezes layout and constrains optimization.

## Derives

Derive standard traits when semantics are obvious.

```rust
// Safe to derive: copy semantics match the byte pattern
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Version(u64);

// Do NOT derive Copy for handles or resources
#[derive(Debug, Clone)] // no Copy — must be explicitly cloned
pub struct ConnectionHandle(NonNull<ffi::Connection>);

// Default only if there's an obvious "empty" value
#[derive(Default)]
pub struct Pagination {
    pub page: u64,         // defaults to 0
    pub page_size: u64,    // defaults to 0
}
```

Do not derive `Copy` for handles, locks, or expensive resources. Do not derive `Default` if no sensible default exists.

## `#[cfg(...)]` Feature Gates

```rust
// Conditionally compile for specific platforms
#[cfg(target_os = "linux")]
fn use_io_uring() { /* ... */ }

#[cfg(not(target_os = "linux"))]
fn use_io_uring() { /* fallback */ }

// Feature-gated modules
#[cfg(feature = "experimental")]
pub mod experimental {
    pub fn new_algorithm() { /* ... */ }
}
```

## Best Practices

- Apply `#[must_use]` to all `Result` and builder types.
- Use `#[non_exhaustive]` on enums in public crates to reserve the right to add variants.
- Feature-gate optional functionality with `#[cfg(feature = "...")]`.
- Use `#[doc(cfg(...))]` (gated behind `cfg(docsrs)`) to document feature requirements.
- Add `#[inline]` only after profiling shows cross-crate inlining is beneficial.
- Prefer `#[repr(transparent)]` for newtype wrappers when no FFI layout is needed.
