# Error Handling in Applications: Anyhow Patterns

Application code focuses on high developer velocity, fast propagation, and rich context reporting. Use `anyhow` to handle diverse errors at application borders.

## 1. Propagation with Context Chaining

Anyhow wraps standard errors automatically and allows injecting contextual breadcrumbs during propagation.

```rust
use anyhow::{Context, Result};
use std::fs::File;

fn load_configs() -> Result<String> {
    let file = File::open("settings.json")
        .context("failed to locate settings.json config file")?;
        
    let config = serde_json::from_reader(file)
        .context("invalid JSON formatting inside settings.json")?;
        
    Ok(config)
}

fn main() {
    if let Err(err) = load_configs() {
        eprintln!("Application Error: {err:?}");
        // Output lists error chain:
        // Application Error: invalid JSON formatting inside settings.json
        // Caused by: failed to locate settings.json config file
    }
}
```

Every `.context()` call adds a frame to the error chain. Use present-tense, actionable messages.

## 2. Downcasting Errors

If you need to inspect an `anyhow::Error` for recovery logic, downcast it to its underlying typed representation.

```rust
use anyhow::Error;

fn handle_app_error(err: Error) {
    if let Some(io_err) = err.downcast_ref::<std::io::Error>() {
        eprintln!("IO error: {}", io_err);
        // Retry or re-queue
    } else if let Some(parse_err) = err.downcast_ref::<serde_json::Error>() {
        eprintln!("Parse error at line {}", parse_err.line());
        // Skip bad record
    } else {
        // Generic fallback — log and abort
        eprintln!("Unexpected error: {:?}", err);
    }
}
```

Downcasting is useful at application boundaries for recovery decisions, but avoid it in library code.

## 3. Bubbling Errors Through Async Boundaries

```rust
use anyhow::{Context, Result};
use tokio::task::JoinHandle;

pub async fn spawn_worker() -> Result<()> {
    let handle: JoinHandle<Result<()>> = tokio::spawn(async {
        process_batch().await
            .context("worker failed processing batch")
    });

    // Re-attach context from the spawned task
    handle.await
        .context("worker panicked or was cancelled")?
        .context("worker returned error")?;

    Ok(())
}
```

Spawned tasks lose their parent context. Use `Instrument` from `tracing` to propagate span context.

## 4. Combining Libraries with anyhow

When application code calls multiple library error types, anyhow unifies them:

```rust
use anyhow::{Context, Result};

async fn process_order(order: Order) -> Result<Invoice> {
    let user = db::find_user(order.user_id)
        .await
        .context("failed to look up user")?;

    let payment = gateway::charge(&user, order.amount)
        .await
        .context("payment gateway rejected transaction")?;

    let invoice = db::create_invoice(user.id, payment.id)
        .await
        .context("failed to persist invoice")?;

    Ok(invoice)
}
```

`sqlx::Error`, `reqwest::Error`, `serde_json::Error`, and `std::io::Error` all unify into `anyhow::Error` automatically.

## 5. Bail Macro for Early Returns

```rust
use anyhow::{bail, Result};

pub fn validate_order(order: &Order) -> Result<()> {
    if order.items.is_empty() {
        bail!("order must contain at least one item");
    }
    if order.amount_minor <= 0 {
        bail!("order amount must be positive, got {}", order.amount_minor);
    }
    Ok(())
}
```

Use `bail!` as a concise alternative to `return Err(anyhow!(...))`.

## 6. Converting Domain Errors to anyhow

```rust
use anyhow::Result;

// A library returns typed errors, but the app wants anyhow.
fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)?; // io::Error → anyhow automatically
    let config: Config = toml::from_str(&content)?; // toml::de::Error → anyhow automatically
    Ok(config)
}
```

All standard `Error` types convert automatically. Only implement `From` when working with non-standard error types.

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

- Use `anyhow::Result` as the return type for `main()`, CLI handlers, and test functions.
- Reserve `thiserror` enums for library boundaries where callers need to pattern-match.
- Add context at every call site where the error meaning changes.
- Don't add context for trivial operations where the error message is already clear.
- Use `bail!` for validation and precondition checks.
- Downcast only at application boundaries for recovery decisions.

## Maintenance Notes

Keep this reference aligned with current Rust idioms. If a crate API changes, update the snippet and mention the version-sensitive part in prose.
