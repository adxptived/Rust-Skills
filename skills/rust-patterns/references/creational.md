# Creational Patterns in Rust

Patterns for object creation and initialization.

## Builder Pattern

### Standard Builder

For types with many optional parameters:

```rust
pub struct HttpRequest {
    url: String,
    method: Method,
    headers: HashMap<String, String>,
    body: Option<Vec<u8>>,
    timeout: Duration,
}

#[derive(Default)]
pub struct HttpRequestBuilder {
    url: Option<String>,
    method: Method,
    headers: HashMap<String, String>,
    body: Option<Vec<u8>>,
    timeout: Option<Duration>,
}

impl HttpRequestBuilder {
    pub fn new() -> Self {
        Self::default()
    }
    
    pub fn url(mut self, url: impl Into<String>) -> Self {
        self.url = Some(url.into());
        self
    }
    
    pub fn method(mut self, method: Method) -> Self {
        self.method = method;
        self
    }
    
    pub fn header(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.headers.insert(key.into(), value.into());
        self
    }
    
    pub fn body(mut self, body: impl Into<Vec<u8>>) -> Self {
        self.body = Some(body.into());
        self
    }
    
    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = Some(timeout);
        self
    }
    
    pub fn build(self) -> Result<HttpRequest, BuildError> {
        let url = self.url.ok_or(BuildError::MissingUrl)?;
        
        Ok(HttpRequest {
            url,
            method: self.method,
            headers: self.headers,
            body: self.body,
            timeout: self.timeout.unwrap_or(Duration::from_secs(30)),
        })
    }
}

// Usage
let request = HttpRequestBuilder::new()
    .url("https://api.example.com/users")
    .method(Method::POST)
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body(r#"{"name": "Alice"}"#)
    .timeout(Duration::from_secs(10))
    .build()?;
```

### Typestate Builder

Compile-time validation of required fields:

```rust
use std::marker::PhantomData;

// State markers
struct NoUrl;
struct HasUrl;
struct NoPort;
struct HasPort;

pub struct ServerBuilder<U, P> {
    url: Option<String>,
    port: Option<u16>,
    workers: usize,
    _markers: PhantomData<(U, P)>,
}

impl ServerBuilder<NoUrl, NoPort> {
    pub fn new() -> Self {
        Self {
            url: None,
            port: None,
            workers: num_cpus::get(),
            _markers: PhantomData,
        }
    }
}

impl<P> ServerBuilder<NoUrl, P> {
    pub fn url(self, url: impl Into<String>) -> ServerBuilder<HasUrl, P> {
        ServerBuilder {
            url: Some(url.into()),
            port: self.port,
            workers: self.workers,
            _markers: PhantomData,
        }
    }
}

impl<U> ServerBuilder<U, NoPort> {
    pub fn port(self, port: u16) -> ServerBuilder<U, HasPort> {
        ServerBuilder {
            url: self.url,
            port: Some(port),
            workers: self.workers,
            _markers: PhantomData,
        }
    }
}

impl<U, P> ServerBuilder<U, P> {
    pub fn workers(mut self, workers: usize) -> Self {
        self.workers = workers;
        self
    }
}

// build() only available when both url and port are set
impl ServerBuilder<HasUrl, HasPort> {
    pub fn build(self) -> Server {
        Server {
            url: self.url.unwrap(),
            port: self.port.unwrap(),
            workers: self.workers,
        }
    }
}

// Usage
let server = ServerBuilder::new()
    .url("localhost")
    .port(8080)
    .workers(4)
    .build();

// Compile error: missing url() or port()
// ServerBuilder::new().build();
// ServerBuilder::new().url("localhost").build();
```

### derive_builder Crate

```rust
use derive_builder::Builder;

#[derive(Builder, Debug)]
#[builder(setter(into))]
pub struct Server {
    host: String,
    #[builder(default = "8080")]
    port: u16,
    #[builder(default)]
    tls: bool,
    #[builder(setter(strip_option), default)]
    max_connections: Option<usize>,
}

// Generated builder
let server = ServerBuilder::default()
    .host("localhost")
    .port(3000)
    .tls(true)
    .max_connections(100)
    .build()?;
```

## Factory Patterns

### Simple Factory Function

```rust
pub enum Database {
    Postgres(PostgresPool),
    Sqlite(SqlitePool),
    Memory(MemoryDb),
}

impl Database {
    pub fn from_url(url: &str) -> Result<Self, DbError> {
        if url.starts_with("postgres://") {
            Ok(Database::Postgres(PostgresPool::connect(url)?))
        } else if url.starts_with("sqlite://") {
            Ok(Database::Sqlite(SqlitePool::connect(url)?))
        } else if url == ":memory:" {
            Ok(Database::Memory(MemoryDb::new()))
        } else {
            Err(DbError::UnsupportedUrl(url.to_string()))
        }
    }
}
```

### Abstract Factory with Traits

```rust
pub trait UIFactory {
    type Button: Button;
    type TextField: TextField;
    type Dialog: Dialog;
    
    fn create_button(&self, label: &str) -> Self::Button;
    fn create_text_field(&self, placeholder: &str) -> Self::TextField;
    fn create_dialog(&self, title: &str) -> Self::Dialog;
}

pub struct MaterialUIFactory;
pub struct CupertinoUIFactory;

impl UIFactory for MaterialUIFactory {
    type Button = MaterialButton;
    type TextField = MaterialTextField;
    type Dialog = MaterialDialog;
    
    fn create_button(&self, label: &str) -> MaterialButton {
        MaterialButton::new(label)
    }
    
    fn create_text_field(&self, placeholder: &str) -> MaterialTextField {
        MaterialTextField::new(placeholder)
    }
    
    fn create_dialog(&self, title: &str) -> MaterialDialog {
        MaterialDialog::new(title)
    }
}

// Usage with generics
fn create_login_form<F: UIFactory>(factory: &F) {
    let username = factory.create_text_field("Username");
    let password = factory.create_text_field("Password");
    let submit = factory.create_button("Login");
}
```

## Singleton Patterns

### OnceLock (Recommended)

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();

pub fn get_config() -> &'static Config {
    CONFIG.get_or_init(|| {
        Config::load_from_env()
            .expect("Failed to load config")
    })
}

// Thread-safe, initialized exactly once
let config = get_config();
```

### LazyLock (Rust 1.80+)

```rust
use std::sync::LazyLock;

static LOGGER: LazyLock<Logger> = LazyLock::new(|| {
    Logger::new()
        .with_level(Level::Info)
        .build()
});

// Automatically initialized on first access
LOGGER.info("Application started");
```

### lazy_static (Legacy)

```rust
use lazy_static::lazy_static;
use std::sync::Mutex;

lazy_static! {
    static ref CACHE: Mutex<HashMap<String, Value>> = {
        Mutex::new(HashMap::new())
    };
}

// Usage
CACHE.lock().unwrap().insert("key".into(), value);
```

## Prototype Pattern

Clone-based object creation:

```rust
pub trait Prototype: Clone {
    fn clone_with_id(&self, id: u64) -> Self;
}

#[derive(Clone)]
pub struct Document {
    id: u64,
    title: String,
    content: String,
    template: DocumentTemplate,
}

impl Prototype for Document {
    fn clone_with_id(&self, id: u64) -> Self {
        let mut clone = self.clone();
        clone.id = id;
        clone
    }
}

// Template registry
pub struct DocumentRegistry {
    templates: HashMap<String, Document>,
}

impl DocumentRegistry {
    pub fn create_from_template(&self, template_name: &str, id: u64) -> Option<Document> {
        self.templates
            .get(template_name)
            .map(|t| t.clone_with_id(id))
    }
}
```

## Default Trait Pattern

```rust
#[derive(Debug)]
pub struct AppConfig {
    pub database_url: String,
    pub port: u16,
    pub workers: usize,
    pub log_level: LogLevel,
    pub features: Features,
}

impl Default for AppConfig {
    fn default() -> Self {
        Self {
            database_url: "postgres://localhost/app".into(),
            port: 8080,
            workers: num_cpus::get(),
            log_level: LogLevel::Info,
            features: Features::default(),
        }
    }
}

// Partial override with struct update syntax
let config = AppConfig {
    port: 3000,
    log_level: LogLevel::Debug,
    ..Default::default()
};
```

## Constructor Conventions

### Multiple Constructors

```rust
pub struct Connection {
    host: String,
    port: u16,
    tls: bool,
}

impl Connection {
    // Primary constructor
    pub fn new(host: impl Into<String>, port: u16) -> Self {
        Self {
            host: host.into(),
            port,
            tls: false,
        }
    }
    
    // Secure variant
    pub fn new_secure(host: impl Into<String>, port: u16) -> Self {
        Self {
            host: host.into(),
            port,
            tls: true,
        }
    }
    
    // From URL
    pub fn from_url(url: &str) -> Result<Self, ParseError> {
        let parsed = Url::parse(url)?;
        Ok(Self {
            host: parsed.host_str().unwrap().into(),
            port: parsed.port().unwrap_or(443),
            tls: parsed.scheme() == "https",
        })
    }
    
    // Fallible constructor
    pub fn try_new(host: &str, port: u16) -> Result<Self, ValidationError> {
        if host.is_empty() {
            return Err(ValidationError::EmptyHost);
        }
        Ok(Self::new(host, port))
    }
}
```

### Into Conversions

```rust
impl From<&str> for Email {
    fn from(s: &str) -> Self {
        Email(s.to_string())
    }
}

impl From<String> for Email {
    fn from(s: String) -> Self {
        Email(s)
    }
}

// TryFrom for fallible conversions
impl TryFrom<&str> for ValidatedEmail {
    type Error = EmailError;
    
    fn try_from(s: &str) -> Result<Self, Self::Error> {
        if s.contains('@') {
            Ok(ValidatedEmail(s.to_string()))
        } else {
            Err(EmailError::Invalid)
        }
    }
}
```
