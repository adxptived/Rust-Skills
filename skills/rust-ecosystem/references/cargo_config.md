# Cargo Workspace & Configuration Patterns

Cargo workspaces allow managing multi-crate monorepos cleanly with shared dependency definitions, profiles, and flags.

## 1. Monorepo Workspaces Layout

Create a root `Cargo.toml` pointing to child folders.

```toml
[workspace]
members = [
    "crates/api-service",
    "crates/shared-types",
    "crates/db-client"
]
resolver = "2"

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.35", features = ["full"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio"] }
```

Inside a child crate, import dependencies dynamically:

```toml
[package]
name = "api-service"
version = "0.1.0"
edition = "2021"

[dependencies]
shared-types = { path = "../shared-types" }
serde = { workspace = true }
tokio = { workspace = true }
```

## 2. Optimizing Cargo Profiles

```toml
[profile.dev]
opt-level = 0
debug = true
split-debuginfo = "unpacked"   # Fast builds on macOS/Windows

[profile.release]
opt-level = 3
lto = "fat"                    # Full link-time optimization
codegen-units = 1              # Maximize per-crate optimization
panic = "abort"                # Remove unwind tables, smaller binary
strip = "symbols"              # Remove debug symbols from release binary

[profile.bench]
opt-level = 3
debug = 1                      # Line-level debug info for profiling
lto = "fat"
```

## 3. Feature Flags

```toml
[features]
default = ["std"]
std = []                    # Enables std-dependent functionality
serde = ["dep:serde"]       # Optional serde support (edition 2021)
native-tls = ["reqwest/native-tls"]
rustls = ["reqwest/rustls"]
```

```rust
// src/lib.rs
#![cfg_attr(not(feature = "std"), no_std)]

#[cfg(feature = "serde")]
use serde::{Serialize, Deserialize};
```

### Feature Unification Rules

- Features are additive — never make a feature remove functionality.
- `default` features are shared across the workspace — a crate can disable them with `default-features = false`.
- Cargo unifies features across the dependency graph — if two crates request different features of the same dependency, both are enabled.

## 4. Conditional Dependencies

```toml
[target.'cfg(target_os = "linux")'.dependencies]
inotify = "0.9"

[target.'cfg(target_os = "windows")'.dependencies]
winapi = { version = "0.3", features = ["fileapi"] }

[target.'cfg(unix)'.dependencies]
nix = "0.27"
```

## 5. Workspace Inheritance

```toml
[workspace.package]
version = "0.1.0"
edition = "2021"
license = "MIT"
repository = "https://github.com/user/repo"

[workspace.dependencies]
thiserror = "1"
serde = { version = "1", default-features = false }

# Child crate:
[package]
name = "my-crate"
version.workspace = true
edition.workspace = true
license.workspace = true
repository.workspace = true
```

## 6. Build Scripts (build.rs)

```rust
// build.rs
fn main() {
    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=config/proto/");

    // Compile protobuf definitions
    prost_build::compile_protos(&["src/schema.proto"], &["src/"]).unwrap();

    // Set compile-time environment variable
    println!("cargo:rustc-env=GIT_HASH={}", git_hash());
}
```

## 7. Cargo Aliases (config.toml)

```toml
# .cargo/config.toml
[alias]
t = "test --all-features"
c = "check --all-targets"
b = "build --release"
w = "watch --exec check --all-targets"
expand = "expand --all-features"
udeps = "udeps --all-targets"
```

## 8. Caching and CI Optimizations

```toml
# .cargo/config.toml
[env]
CARGO_INCREMENTAL = "1"

[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]

[target.'cfg(all())']
# sccache for shared compilation cache
# export RUSTC_WRAPPER=sccache
```

```yaml
# CI: cache Cargo registry and target
actions/cache@v3:
  path: |
    ~/.cargo/registry/
    ~/.cargo/git/
    target/
  key: ${{ runner.os }}-cargo-${{ hashFiles('Cargo.lock') }}
```

## 9. Publishing Best Practices

```toml
[package]
# Required for crates.io
description = "A concise description"
keywords = ["cli", "tool", "utility"]
categories = ["command-line-utilities"]
readme = "README.md"
```

```bash
# Before publishing
cargo publish --dry-run
cargo package --list  # verify what's included
```

Add `[package] exclude = ["ci/", "benches/"]` or use `include` for explicit file lists.

## 10. Cargo.toml Anti-Patterns

- Using `*` version constraints — always specify minimum versions.
- Putting large files in the package — they are bundled with every download.
- Forgetting `default-features = false` when you don't need default features.
- Unifying features unintentionally — use `dep:` syntax in edition 2021 to make features explicit.
- Committing large `target/` caches — use `CARGO_TARGET_DIR` for shared CI caches.
