# Creational Patterns in Rust

Creational patterns focus on constructing valid values while preserving ownership and invariants.

## Constructor Functions

Use `new` for simple, infallible construction and `try_new` when validation can fail.

```rust
pub struct Point { x: f64, y: f64 }

impl Point {
    pub fn new(x: f64, y: f64) -> Self { Self { x, y } }
}
```

## Builder Pattern

Use builders for many optional fields or multi-step validation.

```rust
#[derive(Default)]
pub struct ServerBuilder {
    host: Option<String>,
    port: u16,
}

impl ServerBuilder {
    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }

    pub fn build(self) -> Result<Server, BuildError> {
        let host = self.host.ok_or(BuildError::MissingHost)?;
        Ok(Server { host, port: self.port })
    }
}
```

## Typestate Builder

Use when required fields must be enforced at compile time.

```rust
pub struct Missing;
pub struct Present<T>(T);

pub struct ClientBuilder<Host> {
    host: Host,
}

impl ClientBuilder<Missing> {
    pub fn new() -> Self { Self { host: Missing } }

    pub fn host(self, host: String) -> ClientBuilder<Present<String>> {
        ClientBuilder { host: Present(host) }
    }
}

impl ClientBuilder<Present<String>> {
    pub fn build(self) -> Client {
        Client { host: self.host.0 }
    }
}
```

## Factory Functions

Prefer named constructors over large enum switches in callers.

```rust
impl Transport {
    pub fn tcp(addr: SocketAddr) -> Self { Self::Tcp { addr } }
    pub fn unix(path: PathBuf) -> Self { Self::Unix { path } }
}
```

## Dependency Injection

Pass dependencies as traits/generics instead of creating them internally.

## Smart Constructors for Newtypes

Validate invariants before constructing public values.

## Anti-Patterns

Public fields that bypass validation, `Default` without a meaningful default, builders that panic in `build`, and constructors that perform hidden network or disk IO.
