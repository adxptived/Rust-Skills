---
name: rust-mastery
description: |
  Comprehensive Rust programming skill covering ownership, borrowing, lifetimes, traits, error handling, and idiomatic patterns. ALWAYS use this skill when:
  - User writes, reads, or debugs ANY Rust code (.rs files)
  - User encounters borrow checker errors like "cannot borrow as mutable", "value moved here", "missing lifetime specifier"
  - User asks about ownership, borrowing, references, lifetimes, or the borrow checker
  - User works with Result, Option, ? operator, match expressions, or enums
  - User asks about traits, generics, impl blocks, derive macros, or trait bounds
  - User wants to understand Rust idioms or "the Rust way" to do something
  - User is converting code from another language to Rust
  - User asks about Vec, String, &str, HashMap, iterators, or standard library types
  Even if the user doesn't explicitly say "Rust", trigger this skill for any .rs file or Rust-related compilation error.
---

# Rust Mastery

Comprehensive guide to writing idiomatic, safe, and performant Rust code. This skill synthesizes best practices from "The Rust Programming Language", "Programming Rust", "Effective Rust", and community patterns.

## Quick Navigation

Read additional references based on the task:
- **references/ownership.md** - Deep dive into ownership, borrowing, and lifetimes
- **references/traits.md** - Trait design, standard traits, and generics
- **references/error-handling.md** - Result, Option, and error handling patterns
- **references/idioms.md** - Common Rust idioms and patterns

## Core Principles

### 1. Embrace the Type System

Rust's type system is your ally. Use it to:
- **Express invariants at compile time** - If a state shouldn't exist, make it unrepresentable
- **Encode business logic in types** - Use newtypes, enums, and marker types
- **Leverage zero-cost abstractions** - Generics and traits have no runtime overhead

```rust
// Bad: String that could be anything
fn process_email(email: String) { ... }

// Good: Newtype enforces validation
struct Email(String);
impl Email {
    pub fn new(s: &str) -> Result<Self, EmailError> {
        // Validation happens once, at creation
        if s.contains('@') && s.contains('.') {
            Ok(Self(s.to_string()))
        } else {
            Err(EmailError::Invalid)
        }
    }
}
fn process_email(email: Email) { ... } // Can only receive valid emails
```

### 2. Ownership Fundamentals

Every value in Rust has exactly one owner. When the owner goes out of scope, the value is dropped.

**Three rules to internalize:**
1. Each value has exactly one owner
2. When the owner goes out of scope, the value is dropped
3. Ownership can be transferred (moved) or borrowed (referenced)

```rust
// Move semantics (ownership transfer)
let s1 = String::from("hello");
let s2 = s1; // s1 is moved, no longer valid
// println!("{}", s1); // Compile error!

// Borrowing (reference without taking ownership)
let s1 = String::from("hello");
let len = calculate_length(&s1); // Borrow s1
println!("{} has length {}", s1, len); // s1 still valid

fn calculate_length(s: &str) -> usize {
    s.len()
}
```

### 3. Borrowing Rules

The borrow checker enforces these rules at compile time:
- You can have either ONE mutable reference OR any number of immutable references
- References must always be valid (no dangling references)

```rust
let mut data = vec![1, 2, 3];

// Multiple immutable borrows OK
let r1 = &data;
let r2 = &data;
println!("{:?} {:?}", r1, r2);

// Mutable borrow after immutable borrows end
let r3 = &mut data;
r3.push(4);
```

### 4. Prefer Option and Result Transforms

Avoid explicit `match` when transforms work. This produces cleaner, more composable code.

```rust
// Verbose match-based approach
fn get_user_email(user_id: u32) -> Option<String> {
    match find_user(user_id) {
        Some(user) => match user.email {
            Some(email) => Some(email.to_lowercase()),
            None => None,
        },
        None => None,
    }
}

// Idiomatic transform approach
fn get_user_email(user_id: u32) -> Option<String> {
    find_user(user_id)
        .and_then(|user| user.email)
        .map(|email| email.to_lowercase())
}

// With the ? operator for Results
fn get_user_data(id: u32) -> Result<UserData, Error> {
    let user = find_user(id)?;
    let profile = fetch_profile(&user)?;
    let settings = load_settings(&user)?;
    Ok(UserData { user, profile, settings })
}
```

### 5. Error Handling Philosophy

Rust distinguishes between recoverable errors (`Result`) and unrecoverable errors (`panic!`).

**Use Result for:**
- File operations, network calls, parsing
- Any operation that might legitimately fail
- Library code (let the caller decide how to handle)

**Use panic for:**
- Programming bugs (index out of bounds, unwrap on None)
- Unrecoverable states
- Tests and examples

```rust
// Define custom error types
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataError {
    #[error("Failed to parse data: {0}")]
    ParseError(String),
    #[error("IO error: {0}")]
    IoError(#[from] std::io::Error),
    #[error("Not found: {0}")]
    NotFound(String),
}

// Use anyhow for application code
use anyhow::{Context, Result};

fn process_config() -> Result<Config> {
    let content = std::fs::read_to_string("config.toml")
        .context("Failed to read config file")?;
    let config: Config = toml::from_str(&content)
        .context("Failed to parse config")?;
    Ok(config)
}
```

### 6. Iterator Patterns

Prefer iterator transforms over explicit loops. They're often more readable and equally performant.

```rust
// Imperative style
let mut results = Vec::new();
for item in items {
    if item.is_valid() {
        results.push(item.transform());
    }
}

// Functional style (preferred)
let results: Vec<_> = items
    .iter()
    .filter(|item| item.is_valid())
    .map(|item| item.transform())
    .collect();

// Useful iterator methods
items.iter().find(|x| x.id == target_id)     // First match
items.iter().any(|x| x.is_active())          // Existence check
items.iter().all(|x| x.is_valid())           // Universal check
items.iter().fold(0, |acc, x| acc + x.value) // Reduce/accumulate
items.iter().flat_map(|x| x.children())      // Flatten nested
items.iter().enumerate()                      // Index + value
items.iter().zip(other.iter())               // Pair two iterators
```

### 7. Struct and Enum Design

**Structs for data with named fields:**
```rust
struct User {
    id: UserId,
    name: String,
    email: Email,
    created_at: DateTime<Utc>,
}
```

**Enums for variants/states:**
```rust
enum ConnectionState {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session_id: String },
    Error { message: String, retryable: bool },
}

// Pattern matching extracts data
match state {
    ConnectionState::Connected { session_id } => {
        println!("Session: {}", session_id);
    }
    ConnectionState::Error { message, retryable: true } => {
        println!("Retryable error: {}", message);
    }
    _ => {}
}
```

### 8. Common Patterns

**Builder Pattern for complex construction:**
```rust
#[derive(Default)]
struct RequestBuilder {
    url: Option<String>,
    method: Method,
    headers: HashMap<String, String>,
    timeout: Duration,
}

impl RequestBuilder {
    pub fn new() -> Self {
        Self::default()
    }
    
    pub fn url(mut self, url: impl Into<String>) -> Self {
        self.url = Some(url.into());
        self
    }
    
    pub fn header(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.headers.insert(key.into(), value.into());
        self
    }
    
    pub fn build(self) -> Result<Request, BuildError> {
        let url = self.url.ok_or(BuildError::MissingUrl)?;
        Ok(Request { url, method: self.method, headers: self.headers, timeout: self.timeout })
    }
}

// Usage
let request = RequestBuilder::new()
    .url("https://api.example.com")
    .header("Authorization", "Bearer token")
    .build()?;
```

**Newtype Pattern for type safety:**
```rust
struct UserId(u64);
struct OrderId(u64);

// These cannot be confused, even though both are u64
fn get_user_orders(user_id: UserId) -> Vec<OrderId> { ... }
```

**RAII with Drop:**
```rust
struct TempFile {
    path: PathBuf,
}

impl Drop for TempFile {
    fn drop(&mut self) {
        let _ = std::fs::remove_file(&self.path);
    }
}
// File is automatically deleted when TempFile goes out of scope
```

### 9. Performance Guidelines

- **Clone judiciously** - Cloning is explicit in Rust; use `&str` instead of `String` where possible
- **Use `Cow<str>`** - For functions that might or might not need to allocate
- **Prefer `&[T]`** - Over `&Vec<T>` in function parameters
- **Use `impl Trait`** - For returning iterators without boxing

```rust
use std::borrow::Cow;

// Accepts both owned and borrowed strings efficiently
fn process(input: Cow<'_, str>) -> String {
    if input.contains("special") {
        input.to_uppercase() // Only allocates when needed
    } else {
        input.into_owned()
    }
}

// Flexible slice parameter
fn sum(numbers: &[i32]) -> i32 {
    numbers.iter().sum()
}
// Works with Vec, array, or any contiguous sequence
sum(&vec![1, 2, 3]);
sum(&[1, 2, 3]);
```

### 10. Common Compilation Errors

**E0382: Use of moved value**
```rust
// Error: s moved to s2, then used
let s = String::from("hello");
let s2 = s;
println!("{}", s); // Error!

// Fix: Clone if you need both, or use references
let s = String::from("hello");
let s2 = s.clone();
println!("{} {}", s, s2);
```

**E0502: Cannot borrow as mutable because also borrowed as immutable**
```rust
// Error: Immutable and mutable borrow overlap
let mut v = vec![1, 2, 3];
let first = &v[0];
v.push(4); // Error: v borrowed mutably while first exists
println!("{}", first);

// Fix: End immutable borrow before mutable borrow
let mut v = vec![1, 2, 3];
let first = v[0]; // Copy the value, no borrow
v.push(4);
println!("{}", first);
```

**E0106: Missing lifetime specifier**
```rust
// Error: Rust can't infer lifetime
fn longest(x: &str, y: &str) -> &str { // Error!
    if x.len() > y.len() { x } else { y }
}

// Fix: Add lifetime annotation
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

## Standard Traits to Know

| Trait | Purpose | When to Implement |
|-------|---------|-------------------|
| `Debug` | Debug formatting (`{:?}`) | Always (use `#[derive(Debug)]`) |
| `Clone` | Explicit duplication | When copying makes sense |
| `Copy` | Implicit copy (bitwise) | Small, stack-only types |
| `Default` | Default value | When there's a sensible default |
| `PartialEq`/`Eq` | Equality comparison | Comparable types |
| `PartialOrd`/`Ord` | Ordering | Sortable types |
| `Hash` | Hashing | HashMap/HashSet keys |
| `Display` | User-facing format | Public types |
| `From`/`Into` | Type conversion | When conversion is natural |
| `AsRef`/`AsMut` | Cheap reference conversion | Flexible APIs |
| `Deref` | Smart pointer behavior | Wrapper types |
| `Iterator` | Iteration | Custom collections |
| `Drop` | Cleanup on scope exit | RAII resources |

## Crates to Know

| Category | Crate | Purpose |
|----------|-------|---------|
| Error Handling | `thiserror` | Derive Error for library types |
| Error Handling | `anyhow` | Flexible errors for applications |
| Serialization | `serde` | Serialize/deserialize anything |
| Async Runtime | `tokio` | Async runtime with full features |
| HTTP Client | `reqwest` | Ergonomic HTTP client |
| CLI | `clap` | Command-line argument parsing |
| Logging | `tracing` | Structured logging/tracing |
| Testing | `proptest` | Property-based testing |
| Testing | `criterion` | Benchmarking |

## Quick Reference

See `references/` for deep dives:
- **ownership.md** - Ownership, borrowing, lifetimes in depth
- **traits.md** - Trait system, generics, trait objects
- **error-handling.md** - Complete error handling guide
- **idioms.md** - Idiomatic patterns and anti-patterns
