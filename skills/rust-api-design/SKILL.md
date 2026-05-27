---
name: rust-api-design
description: |
  Rust API Design guidelines, naming conventions, and best practices. Use when:
  - User is designing a library, crate, or public APIs
  - Implementing patterns like Builder, Typestate, or Extension traits
  - Designing public structures, enums, or traits
  - Working with common traits (From/Into, Default, AsRef, Debug, Clone)
  - Applying #[must_use] or #[non_exhaustive] attributes
  - Restricting trait implementation via Sealed Traits
  - Formatting naming structures and lifetime designs
---



## Quick Navigation

- [references/guidelines.md](references/guidelines.md)
- [references/attributes.md](references/attributes.md)

# Rust API Design Guidelines

Expert guide for designing idiomatic, clean, robust, and forward-compatible APIs in Rust.

## Core Rules & Patterns

### 1. Typestate Pattern
Use the type system to enforce valid state transitions at compile time instead of running checks at runtime.

```rust
// States
pub struct Draft;
pub struct Sent;

pub struct Email<State> {
    recipient: String,
    body: String,
    _marker: std::marker::PhantomData<State>,
}

impl Email<Draft> {
    pub fn new(recipient: String, body: String) -> Self {
        Self { recipient, body, _marker: std::marker::PhantomData }
    }

    pub fn send(self) -> Email<Sent> {
        Email {
            recipient: self.recipient,
            body: self.body,
            _marker: std::marker::PhantomData,
        }
    }
}
```

### 2. Builder Pattern
Use builder structs to handle complex creation logic, and mark the build step with `#[must_use]`.

```rust
pub struct Server {
    host: String,
    port: u16,
}

#[derive(Default)]
pub struct ServerBuilder {
    host: Option<String>,
    port: Option<u16>,
}

impl ServerBuilder {
    pub fn host(mut self, host: String) -> Self {
        self.host = Some(host);
        self
    }

    pub fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }

    #[must_use = "builders must be executed to construct the target type"]
    pub fn build(self) -> Result<Server, &'static str> {
        let host = self.host.ok_or("Host required")?;
        let port = self.port.unwrap_or(8080);
        Ok(Server { host, port })
    }
}
```

### 3. Sealed Traits
Prevent downstream crates from implementing public traits by referencing a private marker trait.

```rust
mod private {
    pub trait Sealed {}
}

// Seal the trait by requiring the private Sealed supertrait
pub trait SafeMath: private::Sealed {
    fn safe_add(&self, other: &Self) -> Option<Self> where Self: Sized;
}

// Only implement for trusted types
impl private::Sealed for i32 {}
impl SafeMath for i32 {
    fn safe_add(&self, other: &Self) -> Option<Self> {
        self.checked_add(*other)
    }
}
```

### 4. Extension Traits
Add methods to external types without violating orphan rules.

```rust
pub trait StringExt {
    fn is_alphanumeric_only(&self) -> bool;
}

impl StringExt for String {
    fn is_alphanumeric_only(&self) -> bool {
        self.chars().all(|c| c.is_alphanumeric())
    }
}
```

### 5. Parse, Don't Validate
Encode validity constraints into the type system itself to avoid repeated run-time checks.

```rust
// Bad: validates strings dynamically on every function call
fn process_email(email: &str) { ... }

// Good: parsing parses into a type-safe wrapper representing verified validity
pub struct EmailAddress(String);

impl std::str::FromStr for EmailAddress {
    type Err = &'static str;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        if s.contains('@') {
            Ok(EmailAddress(s.to_string()))
        } else {
            Err("Invalid email format")
        }
    }
}
```

### 6. Attribute Best Practices
- **`#[must_use]`**: Annotate fallible or side-effect-free functions (e.g. builders, getters, calculations) to warn users when their outputs are ignored.
- **`#[non_exhaustive]`**: Annotate public enums and structs to allow adding new variants/fields in patch updates without breaking downstream crates.

### 7. Common Trait Implementations
- **Prefer `From` over `Into`**: Implementing `From<T>` automatically generates `Into<T>`. Only implement `Into` when converting to external types where `From` cannot be implemented due to orphan rules.
- **Implement `Default`**: If a struct provides a parameterless `new()`, implement `Default` to match idiomatic conventions.
- **`AsRef` and `Borrow`**: Use `AsRef<T>` for cheap conversions to reference types, and reserve `Borrow<T>` for cases where the target has matching hash/equality semantics (e.g., hash keys).
- **Derive `Debug`**: Implement or derive `Debug` on all public types to enable trouble-free integration and logging.
- **Derive `Clone` & `PartialEq`** when logical to do so, facilitating tests and standard collection handling.

## Decision Rules

### Public API vs Internal API

Use strict API discipline for anything exported from a crate boundary. Internal modules can move faster, but still keep invariants local.

```rust
// Public: stable, documented, forward-compatible
pub fn connect(config: ConnectConfig) -> Result<Client, ConnectError>;

// Internal: can be narrower and more direct
pub(crate) fn connect_with_parts(addr: SocketAddr, tls: TlsMode) -> io::Result<Socket>;
```

### Generic Parameter vs Trait Object

Use generics for zero-cost static dispatch and trait objects for runtime-selected behavior.

```rust
pub fn encode_with<C: Codec>(codec: C, data: &[u8]) -> Vec<u8> { codec.encode(data) }
pub fn encode_dyn(codec: &dyn Codec, data: &[u8]) -> Vec<u8> { codec.encode(data) }
```

### Builder vs Constructor

Use `new` for required arguments only. Use builders for optional parameters, validation, or future growth.

## Newtype Pattern

```rust
// Strong typing prevents mixing unrelated IDs
pub struct UserId(pub Uuid);
pub struct OrderId(pub Uuid);

// These cannot be accidentally compared or swapped
// But still derive useful traits
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct TenantId(pub Uuid);
```

## Error Type Design

```rust
#[derive(Debug, thiserror::Error)]
pub enum ConnectError {
    #[error("address {addr} is not reachable")]
    Unreachable { addr: SocketAddr },
    #[error("TLS handshake failed")]
    Tls(#[from] rustls::Error),
    #[error("connection timed out after {0:?}")]
    Timeout(Duration),
}

impl ConnectError {
    /// True if the caller should retry.
    pub fn is_retryable(&self) -> bool {
        matches!(self, Self::Unreachable { .. } | Self::Timeout(_))
    }
}
```

## Async API Patterns

```rust
// Good: async fn is clear about being async
pub async fn fetch_user(id: UserId) -> Result<User, Error> { /* ... */ }

// Good: generic over executor
pub async fn run_with<C: Executor>(client: &C) -> Result<(), Error> { /* ... */ }
```

Don't hide blocking work behind `async fn`. Document if the method may block.

## Feature Flag Design

```toml
[features]
default = ["json"]
json = ["serde", "serde_json"]
full = ["json", "grpc", "tls"]
```

- Name features after capabilities, not crate names.
- Keep the default feature set minimal.
- Document feature interactions.

## Naming Conventions

| Pattern | Convention | Example |
|---------|-----------|---------|
| Constructors | `new()` | `Client::new()` |
| Builders | `builder()` | `Client::builder()` |
| Getters | field name | `client.host()` |
| Setters | `set_` prefix | `client.set_host()` |
| Boolean getters | `is_`, `has_`, `can_` | `client.is_connected()` |
| Conversions | `to_` / `into_` / `as_` | `to_string()` / `into_inner()` / `as_ref()` |
| Iterators | plural of item | `users()` returns `impl Iterator<Item = &User>` |

## Production API Checklist

- All public errors are typed and documented.
- Constructors cannot create invalid values.
- Feature-gated APIs show `doc(cfg)`.
- Public enums that may grow use `#[non_exhaustive]`.
- Panic behavior is documented or removed.
- Expensive operations are obvious from the name.
- Async APIs do not hide blocking work.
- Semver implications are considered before exposing fields.
- Newtypes prevent mixing of unrelated domain IDs.
- Builder methods consume and return `Self`.

## Common API Smells

### Leaking Implementation Types

```rust
// Bad: exposes chosen hash map implementation
pub fn users(&self) -> &hashbrown::HashMap<UserId, User>;

// Better: expose behavior
pub fn get_user(&self, id: UserId) -> Option<&User>;
pub fn user_ids(&self) -> impl Iterator<Item = UserId> + '_;
```

### Boolean Parameter Traps

```rust
// Bad
client.connect(true, false);

// Better
client.connect(ConnectOptions { tls: TlsMode::Required, retry: RetryMode::Disabled });
```

### Overusing `String`

Accept `&str` for reading and `impl Into<String>` for storing.

### Leaking `Box<dyn Error>`

```rust
// Bad: callers cannot match on specific errors
pub fn connect() -> Result<(), Box<dyn std::error::Error>>;

// Good: typed errors allow recovery
pub fn connect() -> Result<(), ConnectError>;
```

## Review Prompt

When reviewing Rust API design, inspect naming, ownership, error types, semver stability, feature flags, and whether invalid states can be constructed by safe callers.

## References

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Rust Reference: attributes](https://doc.rust-lang.org/reference/attributes.html)
- [SemVer compatibility in Cargo](https://doc.rust-lang.org/cargo/reference/semver.html)
