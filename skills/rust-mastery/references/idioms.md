# Rust Idioms and Patterns

Common patterns, idioms, and anti-patterns in Rust.

## Table of Contents
1. [Constructors and Builders](#constructors-and-builders)
2. [Type Patterns](#type-patterns)
3. [Memory Patterns](#memory-patterns)
4. [API Design](#api-design)
5. [Anti-Patterns](#anti-patterns)

## Constructors and Builders

### Default Constructor

```rust
#[derive(Default)]
struct Config {
    timeout: Duration,
    retries: u32,
    verbose: bool,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            timeout: Duration::from_secs(30),
            retries: 3,
            verbose: false,
        }
    }
}

// Usage
let config = Config::default();
let config = Config { verbose: true, ..Default::default() };
```

### Builder Pattern

For complex types with optional fields:

```rust
#[derive(Default)]
pub struct ServerBuilder {
    host: Option<String>,
    port: Option<u16>,
    workers: Option<usize>,
}

impl ServerBuilder {
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
    
    pub fn workers(mut self, n: usize) -> Self {
        self.workers = Some(n);
        self
    }
    
    pub fn build(self) -> Result<Server, BuildError> {
        Ok(Server {
            host: self.host.unwrap_or_else(|| "localhost".to_string()),
            port: self.port.ok_or(BuildError::MissingPort)?,
            workers: self.workers.unwrap_or(num_cpus::get()),
        })
    }
}

// Usage
let server = ServerBuilder::new()
    .host("0.0.0.0")
    .port(8080)
    .workers(4)
    .build()?;
```

### TypeState Builder

Enforce required fields at compile time:

```rust
struct NoHost;
struct HasHost(String);

struct ServerBuilder<H> {
    host: H,
    port: u16,
}

impl ServerBuilder<NoHost> {
    pub fn new() -> Self {
        Self { host: NoHost, port: 8080 }
    }
    
    pub fn host(self, host: impl Into<String>) -> ServerBuilder<HasHost> {
        ServerBuilder {
            host: HasHost(host.into()),
            port: self.port,
        }
    }
}

impl ServerBuilder<HasHost> {
    pub fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }
    
    // build() only available when host is set
    pub fn build(self) -> Server {
        Server {
            host: self.host.0,
            port: self.port,
        }
    }
}

// Compile error: can't call build() without host()
// ServerBuilder::new().build();

// OK
let server = ServerBuilder::new()
    .host("localhost")
    .build();
```

## Type Patterns

### Newtype Pattern

Wrap primitive types for type safety:

```rust
struct UserId(u64);
struct OrderId(u64);

// These can't be confused
fn get_user_orders(user_id: UserId) -> Vec<OrderId> { ... }

// With validation
struct Email(String);

impl Email {
    pub fn new(s: impl AsRef<str>) -> Result<Self, EmailError> {
        let s = s.as_ref();
        if s.contains('@') {
            Ok(Self(s.to_string()))
        } else {
            Err(EmailError::InvalidFormat)
        }
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

// Implementing Deref for transparent access
use std::ops::Deref;

impl Deref for Email {
    type Target = str;
    fn deref(&self) -> &str {
        &self.0
    }
}

// Now works: email.len(), email.contains("@"), etc.
```

### State Machine with Enums

```rust
enum Connection {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session: Session },
    Failed { error: Error, retries: u32 },
}

impl Connection {
    fn connect(&mut self) {
        *self = match std::mem::take(self) {
            Connection::Disconnected => {
                Connection::Connecting { attempt: 1 }
            }
            Connection::Failed { retries, .. } => {
                Connection::Connecting { attempt: retries + 1 }
            }
            other => other, // Already connecting or connected
        };
    }
    
    fn on_success(&mut self, session: Session) {
        if matches!(self, Connection::Connecting { .. }) {
            *self = Connection::Connected { session };
        }
    }
}
```

### Phantom Types

Type parameters not stored in struct:

```rust
use std::marker::PhantomData;

struct Meters;
struct Feet;

struct Distance<U> {
    value: f64,
    _unit: PhantomData<U>,
}

impl<U> Distance<U> {
    fn new(value: f64) -> Self {
        Self { value, _unit: PhantomData }
    }
}

// Type-safe: can't add meters and feet
fn add_distances<U>(a: Distance<U>, b: Distance<U>) -> Distance<U> {
    Distance::new(a.value + b.value)
}

let m1: Distance<Meters> = Distance::new(10.0);
let m2: Distance<Meters> = Distance::new(20.0);
let f1: Distance<Feet> = Distance::new(5.0);

add_distances(m1, m2); // OK
// add_distances(m1, f1); // Compile error!
```

## Memory Patterns

### Cow (Clone on Write)

Avoid cloning when borrowing is possible:

```rust
use std::borrow::Cow;

fn process_name(name: Cow<'_, str>) -> Cow<'_, str> {
    if name.contains("bad") {
        // Only allocate when needed
        Cow::Owned(name.replace("bad", "good"))
    } else {
        // Keep borrowed, no allocation
        name
    }
}

process_name(Cow::Borrowed("hello"));        // No allocation
process_name(Cow::Borrowed("hello bad"));    // Allocates
process_name(Cow::Owned(String::from("hi"))); // Already owned
```

### Interior Mutability

When you need mutation through shared reference:

```rust
use std::cell::{Cell, RefCell};
use std::rc::Rc;

// Cell: for Copy types
struct Counter {
    value: Cell<i32>,
}

impl Counter {
    fn increment(&self) {
        self.value.set(self.value.get() + 1);
    }
}

// RefCell: for non-Copy types (runtime borrow checking)
struct Cache {
    data: RefCell<HashMap<String, String>>,
}

impl Cache {
    fn get_or_insert(&self, key: &str, value: &str) -> String {
        let mut data = self.data.borrow_mut();
        data.entry(key.to_string())
            .or_insert_with(|| value.to_string())
            .clone()
    }
}

// Rc<RefCell<T>>: shared mutable ownership
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
let clone1 = Rc::clone(&shared);
let clone2 = Rc::clone(&shared);

clone1.borrow_mut().push(4);
clone2.borrow_mut().push(5);
```

### Entry API

Efficient map access:

```rust
use std::collections::HashMap;

let mut map: HashMap<String, Vec<i32>> = HashMap::new();

// Old way (double lookup)
if !map.contains_key("key") {
    map.insert("key".to_string(), Vec::new());
}
map.get_mut("key").unwrap().push(1);

// Entry API (single lookup)
map.entry("key".to_string())
    .or_insert_with(Vec::new)
    .push(1);

// Entry patterns
map.entry(key).or_default();           // Insert Default::default()
map.entry(key).or_insert(value);       // Insert specific value
map.entry(key).or_insert_with(|| ...); // Compute value lazily
map.entry(key).and_modify(|v| ...);    // Modify if exists
```

## API Design

### Accept Generic Input

```rust
// Bad: requires specific types
fn greet(name: String) { ... }
fn read_file(path: String) { ... }

// Good: accept anything that converts
fn greet(name: impl AsRef<str>) {
    println!("Hello, {}!", name.as_ref());
}

fn read_file(path: impl AsRef<Path>) -> io::Result<String> {
    std::fs::read_to_string(path)
}

// Works with &str, String, &String, PathBuf, &Path, etc.
greet("World");
greet(String::from("Rust"));
read_file("config.toml");
read_file(PathBuf::from("config.toml"));
```

### Return Specific Types

```rust
// Usually return owned types
fn create_user(name: &str) -> User { ... }

// Return references only when borrowing from input
fn first_word(s: &str) -> &str { ... }

// Return impl Trait for iterators
fn evens(v: &[i32]) -> impl Iterator<Item = &i32> {
    v.iter().filter(|n| *n % 2 == 0)
}
```

### Take Ownership When Needed

```rust
// Takes ownership: will store the String
struct User {
    name: String,
}

impl User {
    fn new(name: String) -> Self {
        Self { name }
    }
}

// More flexible: accepts &str or String
impl User {
    fn new(name: impl Into<String>) -> Self {
        Self { name: name.into() }
    }
}

// Usage
User::new("Alice");         // &str converted
User::new(String::from("Bob")); // String moved
```

### Extension Traits

Add methods to external types:

```rust
trait VecExt<T> {
    fn push_if_unique(&mut self, value: T)
    where
        T: PartialEq;
}

impl<T> VecExt<T> for Vec<T> {
    fn push_if_unique(&mut self, value: T)
    where
        T: PartialEq,
    {
        if !self.contains(&value) {
            self.push(value);
        }
    }
}

// Now available on all Vec<T>
let mut v = vec![1, 2, 3];
v.push_if_unique(2); // No change
v.push_if_unique(4); // Adds 4
```

## Anti-Patterns

### Stringly Typed

Bad:
```rust
fn set_state(state: &str) {
    match state {
        "pending" => ...,
        "active" => ...,
        _ => panic!("Invalid state"),
    }
}
```

Good:
```rust
enum State { Pending, Active, Completed }

fn set_state(state: State) {
    match state {
        State::Pending => ...,
        State::Active => ...,
        State::Completed => ...,
    }
}
```

### Clone All The Things

Bad:
```rust
fn process(data: Vec<String>) {
    for item in data.clone() { // Unnecessary clone
        println!("{}", item);
    }
}
```

Good:
```rust
fn process(data: &[String]) {
    for item in data {
        println!("{}", item);
    }
}
```

### Unwrap Everywhere

Bad:
```rust
let file = File::open("config").unwrap();
let content = read_to_string(file).unwrap();
```

Good:
```rust
let file = File::open("config")
    .context("Failed to open config")?;
let content = read_to_string(file)
    .context("Failed to read config")?;
```

### Deref Polymorphism

Using Deref for inheritance-like behavior:

Bad:
```rust
struct Dog {
    animal: Animal,
}

impl Deref for Dog {
    type Target = Animal;
    fn deref(&self) -> &Animal { &self.animal }
}
// Dog::speak() resolves to Animal::speak()
```

Good:
```rust
trait Speak {
    fn speak(&self);
}

impl Speak for Dog {
    fn speak(&self) { println!("Woof!"); }
}
```

### God Objects

Bad:
```rust
struct App {
    users: Vec<User>,
    orders: Vec<Order>,
    products: Vec<Product>,
    // 100 more fields...
    
    fn do_everything(&mut self) { ... }
}
```

Good:
```rust
struct UserService { users: Vec<User> }
struct OrderService { orders: Vec<Order> }
struct ProductService { products: Vec<Product> }

struct App {
    users: UserService,
    orders: OrderService,
    products: ProductService,
}
```
