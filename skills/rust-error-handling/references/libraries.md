# Error Handling in Libraries: Structured API Boundaries

Library code must expose structured, typed errors so that downstream consumers can inspect, match, and recover from failures programmatically.

## 1. Using thiserror for Custom Enums

The `thiserror` crate provides derive macros to construct custom errors without manual boilerplate.

```rust
use thiserror::Error;
use std::path::PathBuf;

#[derive(Error, Debug)]
pub enum ConfigError {
    #[error("configuration file not found at: {0}")]
    NotFound(PathBuf),

    #[error("failed to parse configuration content: {reason}")]
    InvalidFormat {
        reason: String,
        line: usize,
    },

    // Automates conversion from std::io::Error via #[from]
    #[error("system IO failure: {0}")]
    Io(#[from] std::io::Error),
}
```

Each variant gets a user-facing message via `#[error("...")]`. Use `{0}`, `{field}`, or `{field:?}` for interpolation.

## 2. Hiding Internal Failures

Avoid exposing internal dependencies in public error enums. Downstream libraries shouldn't break when you swap internal database drivers.

```rust
// BAD: sqlx::Error is publicly exposed
#[derive(Error, Debug)]
pub enum PublicError {
    #[error("database error: {0}")]
    Db(#[from] sqlx::Error),
}

// GOOD: Map dependency errors to clean domain boundaries
#[derive(Error, Debug)]
pub enum PublicDomainError {
    #[error("item not found in store")]
    ItemNotFound,

    #[error("underlying persistence failure")]
    PersistenceFailure(#[source] sqlx::Error), // Hide detail in source backtrace
}
```

Use `#[source]` when you want to preserve the original error for logging but not expose it in the display message.

## 3. Source Chaining with #[source]

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum JobError {
    #[error("job {job_id} failed validation: {reason}")]
    Validation {
        job_id: String,
        reason: String,
    },

    #[error("job {job_id} could not be processed")]
    Execution {
        job_id: String,
        #[source]
        source: Box<dyn std::error::Error + Send + Sync>,
    },
}

impl JobError {
    pub fn execution(job_id: impl Into<String>, source: impl Into<Box<dyn std::error::Error + Send + Sync>>) -> Self {
        Self::Execution {
            job_id: job_id.into(),
            source: source.into(),
        }
    }
}
```

The `#[source]` attribute marks which field holds the underlying error. When not specified, `thiserror` treats the first field with `#[from]` or a field named `source` as the error source.

## 4. Error Codes for Public APIs

```rust
#[derive(Error, Debug)]
pub enum ApiError {
    #[error("not found")]
    #[error_code("NOT_FOUND")]
    NotFound,

    #[error("rate limit exceeded, retry after {retry_after}s")]
    #[error_code("RATE_LIMITED")]
    RateLimited { retry_after: u64 },

    #[error("service unavailable")]
    #[error_code("SERVICE_UNAVAILABLE")]
    Unavailable { #[source] source: Option<Box<dyn std::error::Error>> },
}

// Manual extension for error_code if you don't use a helper crate.
impl ApiError {
    pub fn code(&self) -> &'static str {
        match self {
            Self::NotFound => "NOT_FOUND",
            Self::RateLimited { .. } => "RATE_LIMITED",
            Self::Unavailable { .. } => "SERVICE_UNAVAILABLE",
        }
    }
}
```

Stable error codes let clients handle errors programmatically without parsing display strings.

## 5. Combining thiserror with anyhow

```rust
use thiserror::Error;

// Library boundary: typed errors.
#[derive(Error, Debug)]
pub enum StorageError {
    #[error("record not found: {0}")]
    NotFound(String),

    #[error("constraint violation: {0}")]
    Constraint(String),

    #[error("connection refused")]
    ConnectionRefused(#[source] Box<dyn std::error::Error + Send + Sync>),
}

// Application code can convert typed errors into anyhow.
use anyhow::{Context, Result};

fn process_record(id: &str) -> Result<()> {
    let record = storage::find(id)
        .map_err(StorageError::NotFound)?
        .context("processing failed")?;
    Ok(())
}
```

Libraries use `thiserror` for typed, matchable errors. Applications use `anyhow` for ergonomic propagation with context.

## 6. Wrapping External Errors

```rust
use std::fmt;

#[derive(Debug)]
pub struct DatabaseError {
    pub message: String,
    pub source: sqlx::Error,
}

impl fmt::Display for DatabaseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "database error: {}", self.message)
    }
}

impl std::error::Error for DatabaseError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        Some(&self.source)
    }
}

impl From<sqlx::Error> for DatabaseError {
    fn from(err: sqlx::Error) -> Self {
        DatabaseError {
            message: "operation failed".to_string(),
            source: err,
        }
    }
}
```

Manual wrapping gives full control over error messages while preserving the source chain.

## Extra Examples

Prefer examples that show both the happy path and the failure path. A good reference snippet should make ownership, errors, and lifecycle boundaries visible.

```rust
fn validate_non_empty(input: &str) -> Result<&str, &'static str> {
    if input.trim().is_empty() {
        Err("input must not be empty")
    } else {
        Ok(input)
    }
}
```

## Best Practices

- Use `thiserror::Error` derive for all public error types in libraries.
- Never implement `Display` or `Error` manually unless you need specific control.
- Hide internal dependency errors behind domain-specific variants.
- Expose stable error codes for programmatic handling.
- Use `#[source]` to preserve the underlying error chain.
- Keep error enums focused: one enum per module or concern, not a global mega-enum.
- Implement `From<InternalError>` for your public error type to keep conversion concise.

## Maintenance Notes

Keep this reference aligned with current Rust idioms. If a crate API changes, update the snippet and mention the version-sensitive part in prose.
