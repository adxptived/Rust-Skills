# Profiling Rust Applications

Deep dive into profiling tools and techniques for Rust.

## CPU Profiling

### perf (Linux)

The gold standard for Linux profiling:

```bash
# Record with call graph
perf record -g --call-graph dwarf target/release/myapp

# Analyze results
perf report

# Flamegraph from perf data
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### flamegraph (Cross-platform)

```bash
cargo install flamegraph

# Profile and generate SVG
cargo flamegraph --bin myapp

# With specific arguments
cargo flamegraph --bin myapp -- --config prod.toml

# Profile tests
cargo flamegraph --test integration_tests

# Profile benchmarks
cargo flamegraph --bench my_benchmark
```

### samply (Cross-platform, macOS-friendly)

Modern profiler with browser UI:

```bash
cargo install samply

# Profile release binary
samply record target/release/myapp

# Opens Firefox Profiler automatically
```

### Instruments (macOS)

```bash
# Time Profiler
xcrun xctrace record --template 'Time Profiler' --launch target/release/myapp

# Allocations
xcrun xctrace record --template 'Allocations' --launch target/release/myapp
```

## Memory Profiling

### DHAT (Heap Profiler)

Valgrind's DHAT ported to Rust:

```toml
[dependencies]
dhat = "0.3"

[features]
dhat-heap = []
```

```rust
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();
    
    // Your application code
    run_app();
}
```

```bash
cargo run --release --features dhat-heap
# Outputs dhat-heap.json - view at https://nnethercote.github.io/dh_view/dh_view.html
```

### Heaptrack (Linux)

```bash
# Install
sudo apt install heaptrack

# Profile
heaptrack target/release/myapp

# Analyze
heaptrack --analyze heaptrack.myapp.*.zst
```

### Valgrind Massif

```bash
valgrind --tool=massif target/release/myapp
ms_print massif.out.*
```

## Allocation Tracking

### Custom Global Allocator

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering};

#[global_allocator]
static ALLOCATOR: CountingAlloc = CountingAlloc;

static ALLOCATED: AtomicUsize = AtomicUsize::new(0);
static FREED: AtomicUsize = AtomicUsize::new(0);
static ALLOC_COUNT: AtomicUsize = AtomicUsize::new(0);

struct CountingAlloc;

unsafe impl GlobalAlloc for CountingAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOCATED.fetch_add(layout.size(), Ordering::Relaxed);
        ALLOC_COUNT.fetch_add(1, Ordering::Relaxed);
        System.alloc(layout)
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        FREED.fetch_add(layout.size(), Ordering::Relaxed);
        System.dealloc(ptr, layout)
    }

    unsafe fn realloc(&self, ptr: *mut u8, layout: Layout, new_size: usize) -> *mut u8 {
        FREED.fetch_add(layout.size(), Ordering::Relaxed);
        ALLOCATED.fetch_add(new_size, Ordering::Relaxed);
        System.realloc(ptr, layout, new_size)
    }
}

fn print_stats() {
    println!("Allocations: {}", ALLOC_COUNT.load(Ordering::Relaxed));
    println!("Allocated: {} bytes", ALLOCATED.load(Ordering::Relaxed));
    println!("Freed: {} bytes", FREED.load(Ordering::Relaxed));
    println!("Current: {} bytes", 
        ALLOCATED.load(Ordering::Relaxed) - FREED.load(Ordering::Relaxed));
}
```

### Tracing Allocator

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::backtrace::Backtrace;

struct TracingAlloc;

unsafe impl GlobalAlloc for TracingAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let ptr = System.alloc(layout);
        if layout.size() > 1024 * 1024 { // Log allocations > 1MB
            eprintln!("Large allocation: {} bytes\n{:?}", 
                layout.size(), Backtrace::capture());
        }
        ptr
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        System.dealloc(ptr, layout)
    }
}
```

## Compile-Time Profiling

### cargo build --timings

```bash
cargo build --release --timings

# Outputs cargo-timing.html with:
# - Crate compilation times
# - Parallelism utilization
# - Critical path analysis
```

### -Z self-profile (Nightly)

```bash
rustup run nightly cargo build -Z self-profile

# Analyze with summarize or crox
cargo install measureme
summarize summarize *.mm_profdata
```

## Benchmarking Best Practices

### Criterion Setup

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "my_benchmark"
harness = false
```

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

fn bench_sorting(c: &mut Criterion) {
    let mut group = c.benchmark_group("sorting");
    
    for size in [100, 1000, 10000] {
        let data: Vec<i32> = (0..size).rev().collect();
        
        group.bench_with_input(
            BenchmarkId::new("std_sort", size),
            &data,
            |b, data| {
                b.iter(|| {
                    let mut d = data.clone();
                    d.sort();
                    black_box(d)
                })
            }
        );
        
        group.bench_with_input(
            BenchmarkId::new("unstable_sort", size),
            &data,
            |b, data| {
                b.iter(|| {
                    let mut d = data.clone();
                    d.sort_unstable();
                    black_box(d)
                })
            }
        );
    }
    
    group.finish();
}

criterion_group!(benches, bench_sorting);
criterion_main!(benches);
```

### Avoiding Benchmark Pitfalls

```rust
// BAD: Compiler might optimize away
fn bench_bad(b: &mut Bencher) {
    b.iter(|| {
        let result = compute_something();
        // Result unused - might be optimized away!
    });
}

// GOOD: Use black_box
fn bench_good(b: &mut Bencher) {
    b.iter(|| {
        black_box(compute_something())
    });
}

// BAD: Setup in iteration
fn bench_bad_setup(b: &mut Bencher) {
    b.iter(|| {
        let data = generate_test_data(); // Slow!
        process(data)
    });
}

// GOOD: Setup outside
fn bench_good_setup(b: &mut Bencher) {
    let data = generate_test_data();
    b.iter(|| {
        process(black_box(&data))
    });
}
```

## Profiling Async Code

### tokio-console

Real-time async task inspection:

```toml
[dependencies]
console-subscriber = "0.2"
tokio = { version = "1", features = ["full", "tracing"] }
```

```rust
#[tokio::main]
async fn main() {
    console_subscriber::init();
    // Your async code
}
```

```bash
# Install console
cargo install tokio-console

# Run your app, then:
tokio-console
```

### Tracing for Async

```rust
use tracing::{instrument, info_span, Instrument};

#[instrument(skip(data))]
async fn process_request(id: u64, data: &[u8]) -> Result<Response, Error> {
    let parsed = parse(data).await?;
    
    let result = async {
        transform(parsed).await
    }
    .instrument(info_span!("transform", id = id))
    .await?;
    
    Ok(result)
}
```

## Quick Reference

| Tool | Platform | Use Case |
|------|----------|----------|
| perf + flamegraph | Linux | CPU profiling |
| samply | Cross-platform | CPU with browser UI |
| Instruments | macOS | CPU, memory, I/O |
| DHAT | Cross-platform | Heap profiling |
| heaptrack | Linux | Memory over time |
| tokio-console | Cross-platform | Async task debugging |
| cargo --timings | Cross-platform | Build time analysis |
