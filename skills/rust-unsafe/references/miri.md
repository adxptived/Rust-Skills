# Miri for Unsafe Rust

Miri interprets Rust programs and catches many classes of undefined behavior.

## Setup

```bash
rustup +nightly component add miri
cargo +nightly miri setup
```

## Run Tests

```bash
cargo +nightly miri test
cargo +nightly miri test --all-features
```

## What Miri Catches

Invalid pointer dereferences, out-of-bounds accesses, use-after-free, alignment violations, some aliasing violations, uninitialized reads, and some data races.

## Limitations

Miri is not a proof of soundness. It explores executed test paths only and may not support all FFI or platform APIs.

## Isolation

Gate Miri-incompatible tests.

```rust
#[cfg_attr(miri, ignore)]
#[test]
fn uses_os_specific_ffi() { /* ... */ }
```

## CI Pattern

```bash
MIRIFLAGS="-Zmiri-strict-provenance" cargo +nightly miri test -p unsafe_core
```


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

## Maintenance Notes

Keep this reference aligned with current Rust idioms. If a crate API changes, update the snippet and mention the version-sensitive part in prose.
