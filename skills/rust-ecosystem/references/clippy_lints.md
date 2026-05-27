# Clippy Linting & Static Code Quality Rules

Clippy analyzes codebase patterns to catch common bugs, efficiency traps, and stylistic issues.

## 1. Configuring Workspace Lints

Configure global lint policies in the root `Cargo.toml` instead of annotating individual source modules.

```toml
[workspace.lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
nursery = { level = "warn", priority = -1 }
clone_on_ref_ptr = "deny"
unwrap_used = "warn"
todo = "warn"
```

## 2. Per-Crate Overrides

```toml
# crates/ffi/Cargo.toml
[lints]
workspace = true

# Allow unsafe in this crate only
[lints.rust]
unsafe_code = "allow"
```

## 3. Essential Clippy Lints

| Lint | Category | Effect |
|------|----------|--------|
| `unwrap_used` | restriction | Prevents panics from `.unwrap()` |
| `expect_used` | restriction | Prevents panics from `.expect()` |
| `panic` | restriction | Prevents `panic!()` calls |
| `unwrap_or_else_default` | pedantic | `unwrap_or_default()` is shorter |
| `redundant_clone` | perf | Detects unnecessary `.clone()` calls |
| `needless_pass_by_value` | perf | Suggests `&T` instead of `T` |
| `large_enum_variant` | style | Flags enums with large size disparities |
| `enum_glob_use` | restriction | Prevents `use Enum::*` |
| `cast_lossless` | correctness | Suggests safe numeric casts |
| `doc_markdown` | style | Backtick-wraps identifiers in docs |
| `missing_panics_doc` | restriction | Requires docs for panicking functions |
| `missing_safety_doc` | correctness | Requires docs for `unsafe` functions |
| `too_many_arguments` | style | Flags functions with 7+ params |

## 4. Clippy Annotation Reference

```rust
// Suppress a lint on a single item
#[allow(clippy::too_many_arguments)]
fn complex(a: i32, b: i32, c: i32, d: i32, e: i32, f: i32, g: i32) {}

// Suppress a lint in an entire module
#![allow(clippy::unwrap_used)]

// Suppress with a reason (stable since Rust 1.67)
#[allow(clippy::cast_possible_truncation)]
fn truncate(value: u64) -> u32 {
    value as u32
}
```

## 5. Running Clippy Effectively

```bash
# Quick check on the whole workspace
cargo clippy --all-targets --all-features

# With warnings as errors (for CI)
cargo clippy --all-targets --all-features -- -D warnings

# Only specific lints
cargo clippy -- -W clippy::pedantic -A clippy::similar_names

# Show lint source inline
cargo clippy --message-format=human
```

## 6. Safety Exemption Overrides

When specific files require different constraints, override lints locally using module attributes.

```rust
// src/ffi.rs
#![allow(clippy::unwrap_used)]

pub fn process_unchecked() {
    let value = Some(42);
    let unwrapped = value.unwrap();
}
```

## 7. Formatting with Rustfmt

Enforce consistent codebase structures by defining a `.rustfmt.toml` in your root workspace.

```toml
# .rustfmt.toml
max_width = 100
tab_spaces = 4
edition = "2021"
use_field_init_shorthand = true
imports_granularity = "Crate"
group_imports = "StdExternalCrate"
reorder_imports = true
match_block_trailing_comma = true
```

### Import Ordering

With `group_imports = "StdExternalCrate"`, rustfmt groups imports as:
1. `std` / `core` / `alloc`
2. External crates
3. `crate` / `super` / `self`

```rust
// Correctly grouped
use std::collections::HashMap;

use axum::{Json, extract::Query};
use serde::Deserialize;

use crate::models::User;
```

## 8. CI Integration

```yaml
# .github/workflows/ci.yml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions-rust-lang/setup-rust-toolchain@v1
        with:
          components: clippy, rustfmt
      - run: cargo clippy --all-targets --all-features -- -D warnings
      - run: cargo fmt --check
```

## 9. Custom Lint Configuration (clippy.toml)

Create `clippy.toml` at workspace root to set thresholds and options:

```toml
# clippy.toml
too-many-arguments-threshold = 8
cognitive-complexity-threshold = 30
disallowed-names = ["foo", "bar", "baz"]  # Force meaningful names
allow-unwrap-in-tests = true
```

## 10. Anti-Patterns

- Silencing lints without a `#[allow(reason = "...")]` comment.
- Adding `#[allow(clippy::all)]` at the crate level — defeats the purpose.
- Running clippy only on `src/` but not `tests/` or `examples/`.
- Not running clippy in CI — local-only linting drifts.
- Fixing clippy warnings by adding `.allow()` instead of fixing root cause.
