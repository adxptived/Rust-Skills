# Error Handling in Rust

Complete guide to handling errors idiomatically with Result, Option, and error types.

## Table of Contents
1. [Philosophy](#philosophy)
2. [Result and Option](#result-and-option)
3. [The ? Operator](#the--operator)
4. [Custom Error Types](#custom-error-types)
5. [Error Handling Crates](#error-handling-crates)
6. [Patterns and Best Practices](#patterns-and-best-practices)

## Philosophy

Rust distinguishes between:
- **Recoverable errors**: `Result<T, E>` - File not found, parse error, network timeout
- **Unrecoverable errors**: `panic!` - Index out of bounds, impossible state

**Guideline**: Library code should return `Result`. Application code decides whether to panic.

## Result and Option

### Option<T>

For values that might not exist:

```rust
fn find_user(id: u32) -> Option<User> {
    users.iter().find(|u| u.id == id).cloned()
}

// Handling Option
let user = find_user(42);

// Pattern matching
match user {
    Some(u) => println!("Found: {}", u.name),
    None => println!("Not found"),
}

// if let
if let Some(u) = user {
    println!("Found: {}", u.name);
}

// Combinators (preferred)
let name = user.map(|u| u.name).unwrap_or_default();
```

### Result<T, E>

For operations that can fail:

```rust
fn read_config() -> Result<Config, io::Error> {
    let content = std::fs::read_to_string("config.toml")?;
    let config = parse_config(&content)?;
    Ok(config)
}

// Handling Result
match read_config() {
    Ok(config) => use_config(config),
    Err(e) => eprintln!("Error: {}", e),
}
```

### Option/Result Methods

| Method | Purpose |
|--------|---------|
| `.unwrap()` | Get value or panic |
| `.expect("msg")` | Get value or panic with message |
| `.unwrap_or(default)` | Get value or use default |
| `.unwrap_or_default()` | Get value or T::default() |
| `.unwrap_or_else(\|\| ...)` | Get value or compute default |
| `.ok()` | Result → Option (discards error) |
| `.err()` | Result → Option<E> (discards value) |
| `.map(f)` | Transform inner value |
| `.map_err(f)` | Transform error |
| `.and_then(f)` | Chain operations (flatMap) |
| `.or_else(f)` | Try alternative on error |
| `.is_some()` / `.is_ok()` | Check state |
| `.as_ref()` | &Option<T> → Option<&T> |
| `.as_mut()` | &mut Option<T> → Option<&mut T> |
| `.transpose()` | Option<Result> ↔ Result<Option> |

### Transform Examples

```rust
// Chaining with map and and_then
fn get_user_email(id: u32) -> Option<String> {
    find_user(id)
        .and_then(|user| user.contact)
        .map(|contact| contact.email)
        .map(|email| email.to_lowercase())
}

// Error transformation
fn fetch_data(url: &str) -> Result<Data, AppError> {
    reqwest::get(url)
        .map_err(AppError::Network)?
        .json()
        .map_err(AppError::Parse)
}

// Filtering
let even: Option<i32> = Some(4).filter(|n| n % 2 == 0);

// Default on None/Err
let value = optional.unwrap_or(42);
let value = result.unwrap_or_else(|e| {
    log::warn!("Using default due to: {}", e);
    default_value()
});
```

## The ? Operator

Propagates errors automatically:

```rust
fn process_file(path: &str) -> Result<Data, Error> {
    let content = std::fs::read_to_string(path)?; // Returns Err early if fails
    let parsed = parse(&content)?;
    let validated = validate(parsed)?;
    Ok(validated)
}

// Equivalent to:
fn process_file_verbose(path: &str) -> Result<Data, Error> {
    let content = match std::fs::read_to_string(path) {
        Ok(c) => c,
        Err(e) => return Err(e.into()),
    };
    // ... same for other operations
}
```

### ? with Option

```rust
fn first_even(numbers: &[i32]) -> Option<i32> {
    let first = numbers.first()?; // Returns None if empty
    if first % 2 == 0 {
        Some(*first)
    } else {
        None
    }
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

## Custom Error Types

### Manual Implementation

```rust
use std::fmt;
use std::error::Error;

#[derive(Debug)]
pub enum AppError {
    NotFound(String),
    InvalidInput { field: String, message: String },
    Internal(Box<dyn Error + Send + Sync>),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::NotFound(id) => write!(f, "Resource not found: {}", id),
            Self::InvalidInput { field, message } => {
                write!(f, "Invalid {}: {}", field, message)
            }
            Self::Internal(e) => write!(f, "Internal error: {}", e),
        }
    }
}

impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            Self::Internal(e) => Some(e.as_ref()),
            _ => None,
        }
    }
}

// From implementations for ? operator
impl From<std::io::Error> for AppError {
    fn from(err: std::io::Error) -> Self {
        AppError::Internal(Box::new(err))
    }
}
```

### thiserror (Recommended for Libraries)

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
    
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

thiserror features:
- `#[error("...")]` - Generates Display
- `#[source]` - Sets Error::source()
- `#[from]` - Generates From impl
- `#[transparent]` - Forwards Display and source()

## Error Handling Crates

### anyhow (Applications)

For application code where error types don't matter to callers:

```rust
use anyhow::{anyhow, bail, Context, Result};

fn process() -> Result<()> {
    // Add context to errors
    let config = read_config()
        .context("Failed to read configuration")?;
    
    // Create ad-hoc errors
    if config.is_empty() {
        bail!("Configuration is empty");
    }
    
    // From any error type
    let data = fetch_data()
        .map_err(|e| anyhow!("Fetch failed: {}", e))?;
    
    Ok(())
}

fn main() {
    if let Err(e) = process() {
        // Print error chain
        eprintln!("Error: {}", e);
        for cause in e.chain().skip(1) {
            eprintln!("Caused by: {}", cause);
        }
        std::process::exit(1);
    }
}
```

### eyre (Enhanced anyhow)

Similar to anyhow with customizable error reports:

```rust
use eyre::{eyre, WrapErr, Result};
use color_eyre::eyre::Report;

fn main() -> Result<()> {
    color_eyre::install()?; // Pretty error reports
    
    process_data().wrap_err("Failed to process data")?;
    Ok(())
}
```

### Choosing Error Crates

| Use Case | Crate |
|----------|-------|
| Library with typed errors | `thiserror` |
| Application (simple) | `anyhow` |
| Application (fancy reports) | `eyre` + `color-eyre` |
| Both library and app | `thiserror` for lib, `anyhow` for app |

## Patterns and Best Practices

### Context with Error Chains

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

// Error output:
// Error: Failed to parse user 42
// Caused by: expected `:` at line 3 column 5
```

### Recoverable vs Fatal

```rust
fn process_batch(items: Vec<Item>) -> Result<Vec<Output>> {
    let mut outputs = Vec::new();
    let mut errors = Vec::new();
    
    for item in items {
        match process_item(item) {
            Ok(output) => outputs.push(output),
            Err(e) => errors.push(e), // Collect, don't fail
        }
    }
    
    if errors.is_empty() {
        Ok(outputs)
    } else {
        // Report all errors, or log and continue
        for e in &errors {
            log::warn!("Item failed: {}", e);
        }
        Ok(outputs) // Partial success
    }
}
```

### Early Return Pattern

```rust
fn validate(input: &Input) -> Result<(), ValidationError> {
    // Fail fast with clear errors
    if input.name.is_empty() {
        return Err(ValidationError::new("name", "cannot be empty"));
    }
    
    if input.age < 0 || input.age > 150 {
        return Err(ValidationError::new("age", "must be between 0 and 150"));
    }
    
    if !input.email.contains('@') {
        return Err(ValidationError::new("email", "invalid format"));
    }
    
    Ok(())
}
```

### Panic Guidelines

When to `panic!`:
- Programming bugs (invariant violations)
- Unrecoverable state
- Tests

When NOT to panic:
- User input errors
- Network/IO errors
- Missing optional data
- Library code (return Result)

```rust
// OK to panic: programming error
fn get_config_value(key: &str) -> &str {
    CONFIG.get(key).expect("Missing required config key")
}

// Return Result: runtime error
fn parse_config(content: &str) -> Result<Config, ParseError> {
    // ...
}
```

### expect vs unwrap

Prefer `expect` over `unwrap` - explains why the panic is "impossible":

```rust
// Bad: no context
let value = map.get(&key).unwrap();

// Good: explains invariant
let value = map.get(&key)
    .expect("key was validated in the previous step");
```

### Option to Result Conversion

```rust
// ok_or: static error
let user = find_user(id).ok_or(UserError::NotFound)?;

// ok_or_else: computed error (lazy)
let user = find_user(id)
    .ok_or_else(|| UserError::NotFound { id })?;
```
