# Unit and Integration Tests

Use the smallest test scope that catches the bug, then add integration coverage for boundaries.

## Unit Tests

Keep unit tests near the code under `#[cfg(test)]`.

```rust
pub fn normalize(input: &str) -> String {
    input.trim().to_lowercase()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn normalize_trims_and_lowercases() {
        assert_eq!(normalize("  Rust "), "rust");
    }
}
```

## Integration Tests

Place black-box tests in `tests/`.

```rust
// tests/api_contract.rs
use my_crate::Client;

#[test]
fn client_builds_with_defaults() {
    let client = Client::builder().build().unwrap();
    assert_eq!(client.timeout().as_secs(), 30);
}
```

## Async Tests

```rust
#[tokio::test]
async fn fetches_user() {
    let user = service.fetch_user(UserId(1)).await.unwrap();
    assert_eq!(user.id, UserId(1));
}
```

## Test Organization Patterns

```text
src/
  lib.rs
  user.rs          # module with inline #[cfg(test)] mod tests
  payment.rs       # module with inline #[cfg(test)] mod tests
tests/
  api_tests.rs     # integration tests — black box, only public API
  db_tests.rs      # integration tests with database
  regression.rs    # regression tests for fixed bugs
benches/
  my_bench.rs      # criterion benchmarks
examples/
  basic_usage.rs   # runnable examples that also serve as smoke tests
```

### Inline test module structure

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_happy_path() { /* ... */ }

    #[test]
    fn test_edge_case() { /* ... */ }

    #[test]
    fn test_error_path() { /* ... */ }
}
```

## Fixtures

Prefer explicit builders over global fixtures.

```rust
fn test_user() -> User {
    User { id: UserId(1), name: "Ada".into(), email: "ada@example.com".into() }
}

fn test_user_builder() -> UserBuilder {
    UserBuilder::default().name("Ada")
}
```

## Assertions

Use precise assertions and include domain context.

```rust
// Weak — doesn't tell you what the actual error was
let result = service.create_user(&data).await;
assert!(result.is_ok());

// Strong — validates the exact result
let user = service.create_user(&data).await.unwrap();
assert_eq!(user.name, "Ada");
assert_eq!(user.email, "ada@example.com");

// Domain error — check the variant
let err = service.create_user(&invalid_data).await.unwrap_err();
assert!(matches!(err, Error::Validation(_)));
```

## Doc Tests

```rust
/// Reverses a string.
///
/// ```
/// use my_crate::reverse;
/// assert_eq!(reverse("hello"), "olleh");
/// ```
pub fn reverse(input: &str) -> String {
    input.chars().rev().collect()
}
```

Doc tests catch API drift — keep them updated and ensure they compile.

## Running Specific Tests

```bash
# Run all tests
cargo test

# Filter by test name
cargo test normalize_

# Run a specific test module
cargo test tests::user::

# Run integration tests only
cargo test --test api_tests

# Include ignored tests
cargo test -- --ignored

# Run with all features
cargo test --all-features

# Show output (non-captured)
cargo test -- --nocapture
```

## Test Organization Anti-Patterns

- Giant `tests/mod.rs` with everything — split by domain.
- Tests that depend on each other (shared mutable state, order-dependent).
- Global fixtures mutated between tests — use fresh creation per test.
- Tests with hidden `unwrap()` calls that should assert specific errors.
- Over-mocking: integration coverage replaced by mock coverage.

## Test Pyramid

```
         ╱╲
        ╱  ╲         E2E / smoke tests (few)
       ╱    ╲
      ╱      ╲      Integration tests (moderate)
     ╱        ╲
    ╱──────────╲    Unit tests (many, fast)
```

Keep the pyramid in mind: most of your test budget goes to fast, scoped unit tests.
