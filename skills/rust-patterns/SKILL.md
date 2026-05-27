---
name: rust-design-patterns
description: |
  Rust design patterns, idioms, and architectural patterns. ALWAYS use this skill when:
  - User wants to design a builder for struct with many optional fields
  - User asks about newtype pattern, type safety, or wrapper types
  - User implements state machines, workflows, or status transitions
  - User asks about typestate pattern or compile-time state validation
  - User wants RAII, Drop trait, or automatic resource cleanup
  - User asks "what's the Rust way to do X" for common patterns
  - User designs APIs, public interfaces, or library ergonomics
  - User asks about extension traits, adding methods to foreign types
  - User wants to avoid common anti-patterns (god objects, stringly typed)
  - User structures a Rust project, modules, or crate organization
  Trigger for ANY "how should I design this in Rust" or "what pattern should I use" questions.
---

# Rust Design Patterns

Collection of idiomatic patterns, architectural choices, and best practices for Rust.

## Quick Navigation
- **references/creational.md** - Builder, Factory, Singleton patterns
- **references/structural.md** - Newtype, Wrapper, Decorator patterns
- **references/behavioral.md** - State machines, Visitor, Strategy

## Idioms

### Default Constructors

```rust
#[derive(Default)]
struct Config {
    timeout: u64,
    retries: u32,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            timeout: 30,
            retries: 3,
        }
    }
}

// Usage
let config = Config::default();
let config = Config { retries: 5, ..Default::default() };
```

### Constructor Functions

```rust
impl User {
    // Primary constructor
    pub fn new(name: String, email: String) -> Self {
        Self { id: Uuid::new_v4(), name, email, created_at: Utc::now() }
    }
    
    // Alternative constructors
    pub fn anonymous() -> Self {
        Self::new("Anonymous".into(), "anonymous@example.com".into())
    }
    
    // Fallible constructor
    pub fn try_new(name: &str, email: &str) -> Result<Self, ValidationError> {
        validate_email(email)?;
        Ok(Self::new(name.into(), email.into()))
    }
}
```

### Destructuring and Ignore

```rust
// Destructure what you need
let Point { x, y: _ } = point;  // Ignore y
let Point { x, .. } = point;    // Ignore rest

// Match with guards
match result {
    Ok(value) if value > 0 => handle_positive(value),
    Ok(_) => handle_zero_or_negative(),
    Err(e) => handle_error(e),
}
```

## Creational Patterns

### Builder Pattern

For complex object construction:

```rust
pub struct ServerConfig {
    host: String,
    port: u16,
    workers: usize,
    tls: Option<TlsConfig>,
}

#[derive(Default)]
pub struct ServerConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    workers: Option<usize>,
    tls: Option<TlsConfig>,
}

impl ServerConfigBuilder {
    pub fn new() -> Self {
        Self::default()
    }
    
    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }
    
    pub fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }
    
    pub fn workers(mut self, workers: usize) -> Self {
        self.workers = Some(workers);
        self
    }
    
    pub fn tls(mut self, tls: TlsConfig) -> Self {
        self.tls = Some(tls);
        self
    }
    
    pub fn build(self) -> Result<ServerConfig, BuildError> {
        Ok(ServerConfig {
            host: self.host.unwrap_or_else(|| "localhost".into()),
            port: self.port.ok_or(BuildError::MissingPort)?,
            workers: self.workers.unwrap_or(num_cpus::get()),
            tls: self.tls,
        })
    }
}

// Usage
let config = ServerConfigBuilder::new()
    .host("0.0.0.0")
    .port(8080)
    .workers(4)
    .build()?;
```

### Typestate Builder

Compile-time enforcement of required fields:

```rust
// Marker types
struct NoHost;
struct HasHost(String);
struct NoPort;
struct HasPort(u16);

struct Builder<H, P> {
    host: H,
    port: P,
    workers: usize,
}

impl Builder<NoHost, NoPort> {
    pub fn new() -> Self {
        Self { host: NoHost, port: NoPort, workers: 4 }
    }
}

impl<P> Builder<NoHost, P> {
    pub fn host(self, host: impl Into<String>) -> Builder<HasHost, P> {
        Builder {
            host: HasHost(host.into()),
            port: self.port,
            workers: self.workers,
        }
    }
}

impl<H> Builder<H, NoPort> {
    pub fn port(self, port: u16) -> Builder<H, HasPort> {
        Builder {
            host: self.host,
            port: HasPort(port),
            workers: self.workers,
        }
    }
}

// build() only available when both host and port are set
impl Builder<HasHost, HasPort> {
    pub fn build(self) -> Server {
        Server {
            host: self.host.0,
            port: self.port.0,
            workers: self.workers,
        }
    }
}

// Compile error: can't call build() without host() and port()
// Builder::new().build();

// OK
let server = Builder::new()
    .host("localhost")
    .port(8080)
    .build();
```

## Structural Patterns

### Newtype Pattern

Type safety through wrapper types:

```rust
// Distinct types from primitives
struct UserId(u64);
struct OrderId(u64);

// Can't accidentally swap them
fn get_user_orders(user_id: UserId) -> Vec<OrderId> { ... }

// With validation
pub struct Email(String);

impl Email {
    pub fn new(s: impl AsRef<str>) -> Result<Self, EmailError> {
        let s = s.as_ref();
        if Self::is_valid(s) {
            Ok(Self(s.to_string()))
        } else {
            Err(EmailError::Invalid)
        }
    }
    
    fn is_valid(s: &str) -> bool {
        s.contains('@') && s.contains('.')
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

// Transparent access via Deref
use std::ops::Deref;

impl Deref for Email {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}

// Now works: email.len(), email.contains("@")
```

### Extension Traits

Add methods to external types:

```rust
pub trait StringExt {
    fn is_blank(&self) -> bool;
    fn truncate_with_ellipsis(&self, max_len: usize) -> String;
}

impl StringExt for str {
    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
    
    fn truncate_with_ellipsis(&self, max_len: usize) -> String {
        if self.len() <= max_len {
            self.to_string()
        } else {
            format!("{}...", &self[..max_len.saturating_sub(3)])
        }
    }
}

// Usage
"   ".is_blank();  // true
"Hello World".truncate_with_ellipsis(8);  // "Hello..."
```

### RAII Guards

Automatic cleanup via Drop:

```rust
pub struct TempFile {
    path: PathBuf,
}

impl TempFile {
    pub fn new(name: &str) -> std::io::Result<Self> {
        let path = std::env::temp_dir().join(name);
        std::fs::File::create(&path)?;
        Ok(Self { path })
    }
    
    pub fn path(&self) -> &Path {
        &self.path
    }
}

impl Drop for TempFile {
    fn drop(&mut self) {
        let _ = std::fs::remove_file(&self.path);
    }
}

// File automatically deleted when TempFile goes out of scope
{
    let temp = TempFile::new("data.tmp")?;
    write_data(temp.path())?;
} // File deleted here
```

### Facade Pattern

Simplify complex subsystems:

```rust
pub struct Database {
    pool: ConnectionPool,
    cache: Cache,
    metrics: Metrics,
}

impl Database {
    pub fn new(config: &Config) -> Result<Self, DbError> {
        Ok(Self {
            pool: ConnectionPool::new(&config.db_url)?,
            cache: Cache::new(config.cache_size),
            metrics: Metrics::new(),
        })
    }
    
    // Simple interface hides complexity
    pub async fn get_user(&self, id: UserId) -> Result<User, DbError> {
        // Check cache first
        if let Some(user) = self.cache.get(&id) {
            self.metrics.record_cache_hit();
            return Ok(user);
        }
        
        // Query database
        let conn = self.pool.get().await?;
        let user = conn.query_one("SELECT * FROM users WHERE id = $1", &[&id]).await?;
        
        // Update cache
        self.cache.set(id, user.clone());
        self.metrics.record_cache_miss();
        
        Ok(user)
    }
}
```

## Behavioral Patterns

### State Machine with Enums

```rust
pub enum ConnectionState {
    Disconnected,
    Connecting { attempt: u32, started_at: Instant },
    Connected { session: Session },
    Disconnecting,
}

pub struct Connection {
    state: ConnectionState,
    config: Config,
}

impl Connection {
    pub fn connect(&mut self) -> Result<(), ConnectionError> {
        self.state = match std::mem::take(&mut self.state) {
            ConnectionState::Disconnected => {
                ConnectionState::Connecting {
                    attempt: 1,
                    started_at: Instant::now(),
                }
            }
            other => return Err(ConnectionError::InvalidState),
        };
        Ok(())
    }
    
    pub fn on_connected(&mut self, session: Session) {
        if matches!(self.state, ConnectionState::Connecting { .. }) {
            self.state = ConnectionState::Connected { session };
        }
    }
    
    pub fn is_connected(&self) -> bool {
        matches!(self.state, ConnectionState::Connected { .. })
    }
}

impl Default for ConnectionState {
    fn default() -> Self {
        Self::Disconnected
    }
}
```

### Typestate Pattern

Encode states in the type system:

```rust
// State types
struct Draft;
struct Published;
struct Archived;

struct Post<State> {
    content: String,
    _state: std::marker::PhantomData<State>,
}

impl Post<Draft> {
    pub fn new(content: String) -> Self {
        Self { content, _state: PhantomData }
    }
    
    pub fn edit(&mut self, content: String) {
        self.content = content;
    }
    
    pub fn publish(self) -> Post<Published> {
        Post { content: self.content, _state: PhantomData }
    }
}

impl Post<Published> {
    pub fn archive(self) -> Post<Archived> {
        Post { content: self.content, _state: PhantomData }
    }
    
    // Can't edit published posts!
}

// Compile-time state enforcement
let post = Post::new("Hello".into());
post.edit("Updated".into());  // OK
let published = post.publish();
// published.edit("Oops".into());  // Compile error!
```

### Strategy Pattern

```rust
trait Compressor {
    fn compress(&self, data: &[u8]) -> Vec<u8>;
    fn decompress(&self, data: &[u8]) -> Vec<u8>;
}

struct GzipCompressor;
struct LzmaCompressor;

impl Compressor for GzipCompressor {
    fn compress(&self, data: &[u8]) -> Vec<u8> { /* ... */ }
    fn decompress(&self, data: &[u8]) -> Vec<u8> { /* ... */ }
}

impl Compressor for LzmaCompressor {
    fn compress(&self, data: &[u8]) -> Vec<u8> { /* ... */ }
    fn decompress(&self, data: &[u8]) -> Vec<u8> { /* ... */ }
}

struct FileProcessor<C: Compressor> {
    compressor: C,
}

impl<C: Compressor> FileProcessor<C> {
    pub fn new(compressor: C) -> Self {
        Self { compressor }
    }
    
    pub fn process(&self, data: &[u8]) -> Vec<u8> {
        self.compressor.compress(data)
    }
}

// Usage
let processor = FileProcessor::new(GzipCompressor);
let compressed = processor.process(&data);
```

### Command Pattern

```rust
trait Command {
    fn execute(&self);
    fn undo(&self);
}

struct InsertText {
    position: usize,
    text: String,
}

impl Command for InsertText {
    fn execute(&self) {
        // Insert text at position
    }
    
    fn undo(&self) {
        // Remove text from position
    }
}

struct CommandHistory {
    commands: Vec<Box<dyn Command>>,
    current: usize,
}

impl CommandHistory {
    pub fn execute(&mut self, cmd: Box<dyn Command>) {
        cmd.execute();
        self.commands.truncate(self.current);
        self.commands.push(cmd);
        self.current += 1;
    }
    
    pub fn undo(&mut self) {
        if self.current > 0 {
            self.current -= 1;
            self.commands[self.current].undo();
        }
    }
    
    pub fn redo(&mut self) {
        if self.current < self.commands.len() {
            self.commands[self.current].execute();
            self.current += 1;
        }
    }
}
```

## Anti-Patterns to Avoid

### God Objects

```rust
// Bad: One type does everything
struct App {
    users: Vec<User>,
    orders: Vec<Order>,
    products: Vec<Product>,
    // ... 50 more fields
    
    fn do_everything(&mut self) { ... }
}

// Good: Separate concerns
struct UserService { ... }
struct OrderService { ... }
struct ProductService { ... }

struct App {
    users: UserService,
    orders: OrderService,
    products: ProductService,
}
```

### Stringly Typed

```rust
// Bad: String for everything
fn set_status(status: &str) {
    match status {
        "pending" | "active" | "done" => { ... }
        _ => panic!("Invalid status"),
    }
}

// Good: Use enums
enum Status { Pending, Active, Done }

fn set_status(status: Status) {
    match status {
        Status::Pending => { ... }
        Status::Active => { ... }
        Status::Done => { ... }
    }
}
```

### Deref Polymorphism

```rust
// Bad: Using Deref for inheritance
struct Dog {
    animal: Animal,
}

impl Deref for Dog {
    type Target = Animal;
    fn deref(&self) -> &Animal { &self.animal }
}

// Good: Use traits
trait Animal {
    fn speak(&self);
}

impl Animal for Dog {
    fn speak(&self) { println!("Woof!"); }
}
```

## Pattern Selection Guide

| Need | Pattern |
|------|---------|
| Complex construction | Builder |
| Compile-time construction validation | Typestate Builder |
| Type safety for primitives | Newtype |
| Add methods to foreign types | Extension Trait |
| Automatic cleanup | RAII Guard |
| Multiple states with transitions | State Machine |
| Compile-time state validation | Typestate |
| Swappable algorithms | Strategy |
| Undoable operations | Command |

## Pattern Checklist

- Name the problem before choosing a pattern.
- Prefer plain functions and enums before trait-object abstractions.
- Use builders when construction has optional fields, validation, or growth pressure.
- Use typestate only when compile-time state prevents real misuse.
- Keep RAII guards small and obvious; document side effects in `Drop`.
- Test pattern examples for both valid paths and misuse they are meant to prevent.

## References

- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [The Rust Programming Language: Traits](https://doc.rust-lang.org/book/ch10-02-traits.html)
