# Profiling Rust Applications

Profiling isolates bottlenecks. Do not optimize from intuition alone.

## Build with Symbols

```toml
[profile.release]
debug = true
lto = "thin"
codegen-units = 1
```

Keep release optimizations enabled while preserving debug symbols.

## CPU Flamegraphs

```bash
cargo install flamegraph
cargo flamegraph --bin my-app
```

Look for wide frames, repeated allocation paths, lock contention, and unexpected formatting/parsing work.

## Linux perf

```bash
perf record -g target/release/my-app
perf report
perf stat -d target/release/my-app
```

## Allocation Profiling

Use heaptrack, DHAT, or platform profilers.

```bash
heaptrack target/release/my-app
heaptrack_gui heaptrack.my-app.*.gz
```

Common allocation sources: `format!` in hot paths, `String` cloning, premature `collect`, unnecessary boxing, and serde intermediate values.

## Benchmarking

Use Criterion for microbenchmarks.

```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_parse(c: &mut Criterion) {
    c.bench_function("parse", |b| b.iter(|| parse_line("a,b,c")));
}

criterion_group!(benches, bench_parse);
criterion_main!(benches);
```

## Async Profiling

For Tokio services, combine CPU profiles with runtime visibility from `tokio-console` and `tracing` spans.

## Regression Workflow

Capture baseline, make one optimization, re-run the same benchmark/profile, and keep only measured wins.

## What to Report

Include workload, hardware, compiler version, profile command, before/after numbers, and confidence interval.
