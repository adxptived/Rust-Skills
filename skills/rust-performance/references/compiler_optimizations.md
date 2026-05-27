# Compiler Optimizations

Compiler settings can improve throughput, binary size, and startup time. Measure each change.

## Release Profile

```toml
[profile.release]
opt-level = 3
codegen-units = 1
lto = "thin"
debug = true
panic = "abort"
```

- `codegen-units = 1` improves cross-module optimization but slows builds.
- `lto = "thin"` is a good default for services and CLIs.
- `panic = "abort"` reduces binary size when unwinding is unnecessary.

## CPU Target

For binaries deployed to known hardware:

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

Do not use `target-cpu=native` for portable release artifacts unless all target machines match.

## Profile-Guided Optimization

```bash
RUSTFLAGS="-Cprofile-generate=/tmp/pgo" cargo build --release
./target/release/my-app realistic-workload
llvm-profdata merge -o /tmp/pgo/merged.profdata /tmp/pgo
RUSTFLAGS="-Cprofile-use=/tmp/pgo/merged.profdata" cargo build --release
```

PGO helps branch-heavy and parser-heavy programs when the training workload matches production.

## Binary Size

```toml
[profile.release-small]
inherits = "release"
opt-level = "z"
lto = true
codegen-units = 1
strip = true
panic = "abort"
```

Use size profiles for CLIs, containers, embedded, or serverless cold starts.

## SIMD

Prefer safe portable abstractions first (`std::simd` when available, `wide`, `packed_simd` alternatives, or domain crates). Use architecture intrinsics only behind feature detection.

```rust
if std::is_x86_feature_detected!("avx2") {
    // call AVX2 implementation
} else {
    // portable fallback
}
```

## Verification

Always compare benchmark results and inspect generated code only after identifying a hot function.
