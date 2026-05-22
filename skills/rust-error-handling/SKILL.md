---
name: rust-error-handling
description: |
  Rust error handling with Result, Option, thiserror, anyhow, and custom error types. ALWAYS use this skill when:
  - User asks about Result<T, E>, Option<T>, or the ? operator
  - User wants to create custom error types or error enums
  - User asks "should I use thiserror or anyhow?"
  - User needs to wrap errors, add context, or chain errors
  - User asks about .unwrap(), .expect(), or when to panic vs return Result
  - User works with .map(), .and_then(), .ok_or(), or other combinators
  - User asks about Error trait, #[from], #[source], or #[error] attributes
  - User building a library and needs typed errors for API consumers
  - User building an application and wants flexible error handling
  - User sees "the trait bound Error is not satisfied" or similar
  Trigger for ANY Rust error handling question, even simple "how do I handle this error".
---

# Rust Error Handling

Complete guide to idiomatic error handling in Rust, from basics to production patterns.

## Philosophy

Rust distinguishes between:
- **Recoverable errors** → `Result<T, E>` (file not found, parse error)
- **Unrecoverable errors** → `panic!` (index out of bounds, invariant violation)

**Key principle**: Return errors, don't panic. Let callers decide how to handle.

## The Basics

### Result<T, E>

```rust
enum Result<T, E> {
    Ok(T),   // Success with value
    Err(E),  // Failure with error
}

fn read_file(path: &str) -> Result<String, std::io::Error> {
    std::fs::read_to_string(path)
}

// Handling
match read_file("config.toml") {
    Ok(content) => println!("{}", content),
    Err(e) => eprintln!("Error: {}", e),
}
```

### Option<T>

```rust
enum Option<T> {
    Some(T),  // Value exists
    None,     // No value
}

fn find_user(id: u32) -> Option<User> {
    users.iter().find(|u| u.id == id).cloned()
}

// Handling
if let Some(user) = find_user(42) {
    println!("Found: {}", user.name);
}
```

## The ? Operator

Propagates errors automatically:

```rust
fn process() -> Result<Data, Error> {
    let content = std::fs::read_to_string("file.txt")?; // Returns Err early
    let parsed = parse(&content)?;
    let validated = validate(parsed)?;
    Ok(validated)
}
```

### ? with Option

```rust
fn first_even(numbers: &[i32]) -> Option<i32> {
    let first = numbers.first()?; // Returns None if empty
    Some(*first)
}
```

### ? in main()

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = read_config()?;
    run_app(config)?;
    Ok(())
}
```

## Combinators

### Option Methods

```rust
// Transform
option.map(|v| v.to_string())           // Option<T> -> Option<U>
option.and_then(|v| other_option(v))    // Flatten nested Option

// Default values
option.unwrap_or(default)               // Value or default
option.unwrap_or_default()              // Value or T::default()
option.unwrap_or_else(|| compute())     // Value or computed

// Conversions
option.ok_or(error)                     // Option -> Result
option.ok_or_else(|| make_error())      // Option -> Result (lazy)

// Filtering
option.filter(|v| v > 0)                // Keep if predicate
option.take()                           // Take value, leave None

// Inspection
option.is_some()                        // Check state
option.is_none()
```

### Result Methods

```rust
// Transform
result.map(|v| v.to_string())           // Result<T, E> -> Result<U, E>
result.map_err(|e| wrap_error(e))       // Result<T, E> -> Result<T, F>
result.and_then(|v| other_result(v))    // Chain Results

// Default values
result.unwrap_or(default)
result.unwrap_or_default()
result.unwrap_or_else(|e| handle(e))

// Conversions
result.ok()                             // Result -> Option (discard err)
result.err()                            // Result -> Option<E>

// Inspection
result.is_ok()
result.is_err()
```

### Chaining Example

```rust
fn get_user_email(id: u32) -> Option<String> {
    find_user(id)
        .and_then(|user| user.contact)
        .map(|contact| contact.email)
        .map(|email| email.to_lowercase())
}

fn fetch_data(url: &str) -> Result<Data, AppError> {
    reqwest::get(url)
        .map_err(AppError::Network)?
        .json()
        .map_err(AppError::Parse)
}
```

## Custom Error Types

### Manual Implementation

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
pub enum AppError {
    NotFound(String),
    InvalidInput { field: String, message: String },
    Io(std::io::Error),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::NotFound(id) => write!(f, "Not found: {}", id),
            Self::InvalidInput { field, message } => {
                write!(f, "Invalid {}: {}", field, message)
            }
            Self::Io(e) => write!(f, "IO error: {}", e),
        }
    }
}

impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            Self::Io(e) => Some(e),
            _ => None,
        }
    }
}

// Enable ? operator conversions
impl From<std::io::Error> for AppError {
    fn from(err: std::io::Error) -> Self {
        AppError::Io(err)
    }
}
```

## thiserror (Libraries)

Derive macro for error types. Best for libraries with typed errors.

```toml
[dependencies]
thiserror = "2"
```

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataError {
    #[error("Failed to read file: {path}")]
    ReadError {
        path: String,
        #[source]
        source: std::io::Error,
    },
    
    #[error("Parse error at line {line}: {message}")]
    ParseError { line: usize, message: String },
    
    #[error("Validation failed: {0}")]
    ValidationError(String),
    
    // Transparent: forwards Display and source()
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

### Attributes

| Attribute | Effect |
|-----------|--------|
| `#[error("...")]` | Generates Display impl |
| `#[source]` | Sets Error::source() |
| `#[from]` | Generates From impl |
| `#[transparent]` | Forwards Display/source |

## anyhow (Applications)

Flexible error handling for application code.

```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::{anyhow, bail, Context, Result};

fn process() -> Result<()> {
    // Add context to existing errors
    let config = read_config()
        .context("Failed to read configuration")?;
    
    // Create ad-hoc errors
    if config.is_empty() {
        bail!("Configuration is empty");
    }
    
    // Convert from any error
    let data = fetch_data()
        .map_err(|e| anyhow!("Fetch failed: {}", e))?;
    
    Ok(())
}

fn main() {
    if let Err(e) = process() {
        // Print full error chain
        eprintln!("Error: {:#}", e);
        
        // Or iterate chain
        for cause in e.chain() {
            eprintln!("  Caused by: {}", cause);
        }
        
        std::process::exit(1);
    }
}
```

### anyhow Features

```rust
// Downcast to concrete type
if let Some(io_err) = e.downcast_ref::<std::io::Error>() {
    // Handle IO error specifically
}

// Root cause
let root = e.root_cause();

// Attach context
let result = operation()
    .with_context(|| format!("Failed for user {}", user_id))?;
```

## Library vs Application

### Library Code → thiserror

```rust
// lib.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyLibError {
    #[error("Invalid input: {0}")]
    InvalidInput(String),
    #[error("Not found")]
    NotFound,
    #[error(transparent)]
    Io(#[from] std::io::Error),
}

pub fn process(input: &str) -> Result<Output, MyLibError> {
    // Return typed errors - let users handle them
}
```

### Application Code → anyhow

```rust
// main.rs
use anyhow::{Context, Result};
use my_lib::{process, MyLibError};

fn main() -> Result<()> {
    let result = process("input")
        .context("Processing failed")?;
    
    // Application decides what to do with errors
    Ok(())
}
```

## Error Patterns

### Wrapping External Errors

```rust
#[derive(Error, Debug)]
pub enum ServiceError {
    #[error("Database error")]
    Database(#[source] sqlx::Error),
    
    #[error("HTTP error")]
    Http(#[source] reqwest::Error),
}

impl From<sqlx::Error> for ServiceError {
    fn from(e: sqlx::Error) -> Self {
        ServiceError::Database(e)
    }
}
```

### Error Context

```rust
use anyhow::Context;

fn load_user(id: u32) -> Result<User> {
    let path = format!("/data/users/{}.json", id);
    
    let content = std::fs::read_to_string(&path)
        .with_context(|| format!("Failed to read user file: {}", path))?;
    
    let user: User = serde_json::from_str(&content)
        .with_context(|| format!("Failed to parse user {}", id))?;
    
    Ok(user)
}
```

### Multiple Error Types

```rust
// When function can return different error types
fn complex_operation() -> Result<Data, Box<dyn Error>> {
    let file = std::fs::read_to_string("data.json")?; // io::Error
    let data: Data = serde_json::from_str(&file)?;    // serde error
    Ok(data)
}

// Or use anyhow
fn complex_operation() -> anyhow::Result<Data> {
    let file = std::fs::read_to_string("data.json")?;
    let data: Data = serde_json::from_str(&file)?;
    Ok(data)
}
```

### Recoverable Errors

```rust
fn process_batch(items: Vec<Item>) -> (Vec<Output>, Vec<Error>) {
    let mut successes = Vec::new();
    let mut failures = Vec::new();
    
    for item in items {
        match process_item(item) {
            Ok(output) => successes.push(output),
            Err(e) => failures.push(e),
        }
    }
    
    (successes, failures)
}
```

## When to Panic

**Panic for:**
- Programming bugs (violated invariants)
- Impossible states
- Setup failures (can't recover anyway)
- Tests

**Don't panic for:**
- User input errors
- Network/IO errors
- Missing optional data
- Library code

```rust
// OK to panic: invariant violation
fn get_item(index: usize) -> &Item {
    self.items.get(index)
        .expect("index validated in new()")
}

// Don't panic: runtime error
fn parse_config(content: &str) -> Result<Config, ParseError> {
    // Return Result, let caller decide
}
```

## unwrap vs expect

```rust
// Bad: no context
let value = option.unwrap();

// Better: explains invariant
let value = option.expect("value set during initialization");

// Best: handle the error
let value = option.ok_or(Error::MissingValue)?;
```

## Quick Reference

| Crate | Use For | Error Type |
|-------|---------|------------|
| thiserror | Libraries | Typed enum |
| anyhow | Applications | `anyhow::Error` |
| eyre | Apps (fancy output) | `eyre::Report` |

| Method | Effect |
|--------|--------|
| `?` | Propagate error |
| `.context("msg")` | Add context (anyhow) |
| `.ok_or(err)` | Option → Result |
| `.map_err(f)` | Transform error |
| `.unwrap_or(v)` | Default value |
