# Rust API Guidelines

Design APIs that are unsurprising, hard to misuse, and stable across releases.

## Naming

- Types and traits: `UpperCamelCase` (`HttpClient`, `ParseError`).
- Functions, methods, modules, fields: `snake_case`.
- Constructors: prefer `new`, `with_*`, `from_*`, `try_new`.
- Fallible APIs: use `try_` only when there is also an infallible counterpart.
- Conversions: implement `From<T>` for infallible conversions and `TryFrom<T>` for fallible ones.

```rust
pub struct Email(String);

impl TryFrom<String> for Email {
    type Error = EmailError;

    fn try_from(value: String) -> Result<Self, Self::Error> {
        validate_email(&value)?;
        Ok(Self(value))
    }
}
```

## Ownership in Parameters

Accept the least restrictive type that still expresses intent.

```rust
fn parse_path(path: impl AsRef<std::path::Path>) -> Result<Config, Error>;
fn set_name(name: impl Into<String>);
fn write_all(bytes: impl AsRef<[u8]>);
```

Use borrowed values for read-only access, owned values when storing.

## Return Types

- Return concrete types from functions unless abstraction is required.
- Return `impl Trait` for opaque iterators/futures.
- Use trait objects for runtime polymorphism.

```rust
pub fn names(&self) -> impl Iterator<Item = &str> {
    self.users.iter().map(|u| u.name.as_str())
}
```

## Error Surface

Libraries should expose typed errors. Applications may use `anyhow` at boundaries.

```rust
#[derive(Debug, thiserror::Error)]
pub enum ParseConfigError {
    #[error("missing required field `{0}`")]
    MissingField(&'static str),
    #[error("invalid port `{0}`")]
    InvalidPort(u16),
}
```

## Builders

Use builders when construction has many optional fields or validation.

```rust
#[must_use]
pub struct ClientBuilder {
    timeout: std::time::Duration,
    retries: usize,
}

impl ClientBuilder {
    pub fn timeout(mut self, timeout: std::time::Duration) -> Self {
        self.timeout = timeout;
        self
    }

    pub fn build(self) -> Result<Client, BuildError> {
        if self.timeout.is_zero() {
            return Err(BuildError::ZeroTimeout);
        }
        Ok(Client { timeout: self.timeout, retries: self.retries })
    }
}
```

## Extensibility

Use `#[non_exhaustive]` on public enums/structs when future fields or variants are likely.

```rust
#[non_exhaustive]
pub enum Event {
    Connected,
    Disconnected,
}
```

## Semver Safety

Breaking changes include removing public items, changing trait method signatures, adding required trait methods, making public fields private, narrowing accepted input types, or adding enum variants without `#[non_exhaustive]`.

Prefer additive changes and default trait methods.

## Documentation

Document invariants, panics, errors, examples, and feature flags.
