---
name: rust-ecosystem
description: |
  Rust ecosystem — crate selection, Cargo features, workspaces, and tooling. Use this skill when:
  - User asks "which crate should I use for X?"
  - User works with Cargo workspaces or multi-crate projects
  - User asks about feature flags, optional dependencies, or cfg attributes
  - User asks about Cargo.lock, semver, or dependency management
  - User sets up CI for a Rust project
  - User asks about cargo plugins (cargo-audit, cargo-deny, cargo-expand, etc.)
  - User publishes a crate to crates.io
  - User asks about linking to C libraries (bindgen, cc crate)
  - User asks about proc macros or derive macros
  Trigger when the question is about crates, tooling, or project configuration.
---



## Quick Navigation

- [references/cargo_config.md](references/cargo_config.md)
- [references/clippy_lints.md](references/clippy_lints.md)

# Rust Ecosystem

The essential crates, tools, and patterns for productive Rust development.

## Crate Selection Guide

### Serialization
| Need | Crate |
|------|-------|
| JSON / YAML / TOML (derive) | `serde` + format crate |
| JSON | `serde_json` |
| TOML | `toml` |
| YAML | `serde_yaml` |
| MessagePack (binary) | `rmp-serde` |
| Protocol Buffers | `prost` |
| Fast binary | `bincode`, `bitcode` |

### HTTP & Networking
| Need | Crate |
|------|-------|
| HTTP client (async) | `reqwest` |
| HTTP client (sync) | `ureq` |
| Web framework | `axum`, `actix-web` |
| Low-level HTTP | `hyper` |
| gRPC | `tonic` |
| WebSocket | `tokio-tungstenite` |

### Async
| Need | Crate |
|------|-------|
| Async runtime | `tokio` (full), `smol` (light) |
| Async traits | `async-trait` |
| Streams | `futures`, `tokio-stream` |
| Async utilities | `futures-util` |

### Databases
| Need | Crate |
|------|-------|
| PostgreSQL / MySQL / SQLite (async) | `sqlx` |
| ORM | `diesel` |
| Redis | `redis` |
| MongoDB | `mongodb` |
| SQLite (sync) | `rusqlite` |

### Error Handling
| Need | Crate |
|------|-------|
| Library errors | `thiserror` |
| Application errors | `anyhow` |
| Fancy error output | `miette` |

### CLI
| Need | Crate |
|------|-------|
| Argument parsing | `clap` (derive) |
| Progress bars | `indicatif` |
| Terminal color | `colored`, `nu-ansi-term` |
| TUI | `ratatui` + `crossterm` |
| Terminal detect | `atty`, `is-terminal` |

### Data & Algorithms
| Need | Crate |
|------|-------|
| Faster HashMap | `rustc-hash`, `ahash` |
| SmallVec | `smallvec`, `tinyvec` |
| Concurrent data structures | `dashmap`, `crossbeam` |
| UUID | `uuid` |
| Dates/times | `chrono`, `time` |
| Regular expressions | `regex` |
| Random numbers | `rand` |

### Observability
| Need | Crate |
|------|-------|
| Structured logging | `tracing` |
| Log facade | `log` |
| tracing subscriber | `tracing-subscriber` |
| OpenTelemetry | `opentelemetry`, `tracing-opentelemetry` |

## Cargo Workspaces

For multi-crate projects, use workspaces to share dependencies and build settings:

```toml
# workspace Cargo.toml
[workspace]
members = ["crates/core", "crates/api", "crates/cli", "crates/worker"]
resolver = "2"  # Use this

# Share dependency versions across workspace
[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
anyhow = "1"
tracing = "0.1"
sqlx = { version = "0.8", features = ["postgres", "runtime-tokio-rustls"] }
```

```toml
# crates/api/Cargo.toml
[dependencies]
# Inherit version from workspace
tokio.workspace = true
serde.workspace = true
# Override features if needed
serde = { workspace = true, features = ["derive", "rc"] }
# Crate-specific dep
axum = "0.8"
```

```toml
# Workspace lint configuration
[workspace.lints.rust]
unsafe_code = "forbid"

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
# Specific overrides
module_name_repetitions = "allow"
```

## Feature Flags

```toml
[features]
default = ["std"]
std = []
async = ["dep:tokio", "dep:futures"]
full = ["async", "tls"]
tls = ["dep:rustls", "dep:tokio-rustls"]

[dependencies]
tokio = { version = "1", optional = true }
rustls = { version = "0.23", optional = true }
tokio-rustls = { version = "0.26", optional = true }
```

```rust
// Conditional compilation
#[cfg(feature = "async")]
pub async fn fetch(url: &str) -> Result<String, Error> {
    reqwest::get(url).await?.text().await.map_err(Into::into)
}

#[cfg(not(feature = "std"))]
extern crate alloc; // no_std with alloc

// cfg_if for cleaner feature branches
use cfg_if::cfg_if;
cfg_if! {
    if #[cfg(feature = "tls")] {
        use rustls::ClientConfig;
        fn make_connector() -> ClientConnector { ClientConnector::new() }
    } else {
        fn make_connector() -> PlainConnector { PlainConnector::new() }
    }
}
```

## Cargo Profile Configuration

```toml
[profile.dev]
opt-level = 0      # No optimization (fast compile)
debug = true       # Debug symbols
overflow-checks = true

[profile.release]
opt-level = 3      # Full optimization
lto = true         # Link-time optimization (slower compile, smaller/faster binary)
codegen-units = 1  # Single codegen unit for best optimization
panic = "abort"    # Smaller binary (no unwinding)
strip = true       # Strip symbols (add strip = "debuginfo" to keep some)

# For profiling: release speed + debug symbols
[profile.profiling]
inherits = "release"
debug = true
strip = false

# For benchmarks
[profile.bench]
opt-level = 3
lto = true
```

## Essential Cargo Plugins

```bash
# Security audit
cargo install cargo-audit
cargo audit

# Deny specific licenses/advisories
cargo install cargo-deny
cargo deny check

# Expand macros (debug derive, etc.)
cargo install cargo-expand
cargo expand -- src/models.rs

# Faster linking (dev builds)
# Linux: use lld or mold
# Cargo.toml or .cargo/config.toml:
# [target.x86_64-unknown-linux-gnu]
# linker = "clang"
# rustflags = ["-C", "link-arg=-fuse-ld=mold"]

# Unused dependencies
cargo install cargo-udeps
cargo +nightly udeps

# Outdated dependencies
cargo install cargo-outdated
cargo outdated

# Benchmark/test with coverage
cargo install cargo-llvm-cov
cargo llvm-cov --html

# Binary size analysis
cargo install cargo-bloat
cargo bloat --release

# Watch mode (recompile on save)
cargo install cargo-watch
cargo watch -x test
```

## Build Scripts (build.rs)

```rust
// build.rs — runs before compilation
fn main() {
    // Rerun only if these change
    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=proto/");
    println!("cargo:rerun-if-env-changed=MY_API_KEY");

    // Link a C library
    println!("cargo:rustc-link-lib=ssl");
    println!("cargo:rustc-link-search=/usr/lib");

    // Set a cfg flag
    if cfg!(target_os = "windows") {
        println!("cargo:rustc-cfg=windows_target");
    }

    // Generate code at build time
    let out_dir = std::env::var("OUT_DIR").unwrap();
    let dest = std::path::Path::new(&out_dir).join("generated.rs");
    std::fs::write(dest, "pub const BUILD_DATE: &str = \"2025-01-01\";").unwrap();
}
```

```rust
// main.rs: include generated code
include!(concat!(env!("OUT_DIR"), "/generated.rs"));
```

## Proc Macros (Custom Derives)

```toml
# my-macros/Cargo.toml
[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

```rust
// my-macros/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(MyTrait)]
pub fn derive_my_trait(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;

    let expanded = quote! {
        impl MyTrait for #name {
            fn name(&self) -> &str {
                stringify!(#name)
            }
        }
    };

    TokenStream::from(expanded)
}
```

## Publishing to crates.io

```toml
[package]
name = "my-crate"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <you@example.com>"]
description = "A short description"
license = "MIT OR Apache-2.0"  # Standard Rust dual license
repository = "https://github.com/you/my-crate"
homepage = "https://my-crate.rs"
documentation = "https://docs.rs/my-crate"
keywords = ["keyword1", "keyword2"]  # Max 5, searchable
categories = ["data-structures"]    # From allowed list
readme = "README.md"
exclude = ["tests/", "benches/", ".github/"]
```

```bash
cargo login                  # Authenticate
cargo package --list         # Preview what will be published
cargo publish --dry-run      # Dry run
cargo publish                # Publish
```

## CI Configuration (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2  # Cache build artifacts
      - run: cargo test --all-features --workspace

  clippy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - run: cargo clippy --all-targets --all-features -- -D warnings

  fmt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --check

  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rustsec/audit-check@v2
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

## Anti-Patterns

```toml
# Bad: default features silently pull in async runtimes, TLS stacks, or native deps.
serde = "1"
reqwest = "0.12"

# Good: choose feature surface explicitly.
serde = { version = "1", features = ["derive"] }
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }
```

```bash
# Bad: publish without checking packaged contents.
cargo publish

# Good: inspect and dry-run first.
cargo package --list
cargo publish --dry-run
```

## Maintenance Checklist

- Pin MSRV in docs and CI if the crate promises one.
- Audit default features for transitive dependency and compile-time impact.
- Run `cargo tree -d` before adding crates that may duplicate versions.
- Run `cargo publish --dry-run` and inspect `cargo package --list`.
- Use workspace-level dependencies for shared versions.
- Keep clippy, fmt, test, audit, and docs checks in CI.

## References

- [lib.rs](https://lib.rs) — crate discovery
- [blessed.rs](https://blessed.rs) — curated crate recommendations
- [Cargo reference](https://doc.rust-lang.org/cargo/)
- [The rustup book](https://rust-lang.github.io/rustup/)
