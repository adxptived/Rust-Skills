---
name: rust-performance
description: |
  Rust performance optimization, profiling, and benchmarking skill. ALWAYS use this skill when:
  - User says Rust code is "slow", "takes too long", or needs to be "faster"
  - User wants to profile CPU usage, memory allocations, or find bottlenecks
  - User asks about flamegraph, perf, criterion, DHAT, heaptrack, or any profiling tool
  - User is concerned about allocations, cloning, or memory efficiency
  - User asks about Vec::with_capacity, Cow, SmallVec, or allocation strategies
  - User wants to write benchmarks or compare performance of implementations
  - User asks about release mode, LTO, codegen-units, or compiler optimizations
  - User mentions "too many allocations", "memory usage", or "throughput"
  - User works with large datasets, parsing, or performance-critical code
  Even for general "make this faster" requests on Rust code, use this skill.
---

# Rust Performance Optimization

Comprehensive guide to profiling, measuring, and optimizing Rust code. Based on "The Rust Performance Book" and real-world optimization patterns.

## Quick Navigation
- **references/profiling.md** - Profiling tools and techniques
- **references/allocations.md** - Reducing heap allocations
- **references/concurrency.md** - Parallel and async optimization

## Golden Rules

1. **Measure first** - Profile before optimizing
2. **Optimize hot paths** - 90% of time in 10% of code
3. **Benchmark changes** - Verify improvements
4. **Consider tradeoffs** - Speed vs memory vs complexity

## Release Mode

Always benchmark in release mode with optimizations:

```toml
# Cargo.toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
panic = "abort"

# For profiling (symbols but optimized)
[profile.release-with-debug]
inherits = "release"
debug = true
```

```bash
cargo build --release
cargo run --release
```

## Quick Performance Wins

### 1. Use `&str` Instead of `String`

```rust
// Allocates on every call
fn greet_slow(name: String) {
    println!("Hello, {}!", name);
}

// Zero allocation
fn greet_fast(name: &str) {
    println!("Hello, {}!", name);
}
```

### 2. Pre-allocate Collections

```rust
// Reallocates as it grows
let mut v = Vec::new();
for i in 0..1000 {
    v.push(i);
}

// Single allocation
let mut v = Vec::with_capacity(1000);
for i in 0..1000 {
    v.push(i);
}

// Even better: use iterators
let v: Vec<_> = (0..1000).collect();
```

### 3. Avoid Unnecessary Clones

```rust
// Bad: clones unnecessarily
fn process(items: Vec<Item>) -> Vec<Result> {
    items.iter()
        .map(|item| item.clone())  // Don't clone if you don't need it
        .map(|item| transform(item))
        .collect()
}

// Good: take ownership or borrow
fn process(items: Vec<Item>) -> Vec<Result> {
    items.into_iter()  // Consume the Vec
        .map(transform)
        .collect()
}
```

### 4. Use `Cow<str>` for Maybe-Owned Strings

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_"))
    } else {
        Cow::Borrowed(input)  // No allocation
    }
}
```

### 5. Use `collect()` Strategically

```rust
// Creates intermediate Vec
let sum: i32 = items.iter()
    .map(|x| x * 2)
    .collect::<Vec<_>>()  // Unnecessary
    .iter()
    .sum();

// Direct iteration
let sum: i32 = items.iter()
    .map(|x| x * 2)
    .sum();
```

## Benchmarking

### Criterion.rs

The gold standard for Rust benchmarks:

```toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "my_benchmark"
harness = false
```

```rust
// benches/my_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 | 1 => n,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn criterion_benchmark(c: &mut Criterion) {
    c.bench_function("fib 20", |b| {
        b.iter(|| fibonacci(black_box(20)))
    });
    
    // Compare implementations
    let mut group = c.benchmark_group("String Ops");
    group.bench_function("clone", |b| {
        b.iter(|| String::from("hello").clone())
    });
    group.bench_function("to_string", |b| {
        b.iter(|| "hello".to_string())
    });
    group.finish();
}

criterion_group!(benches, criterion_benchmark);
criterion_main!(benches);
```

```bash
cargo bench
cargo bench -- "fib"  # Run specific benchmarks
```

### Key Points
- Use `black_box()` to prevent optimization
- Multiple iterations for statistical significance
- Compare baseline vs optimized
- Watch for outliers

## Profiling

### CPU Profiling

**perf (Linux):**
```bash
perf record -g --call-graph dwarf target/release/myapp
perf report
```

**flamegraph:**
```bash
cargo install flamegraph
cargo flamegraph --bin myapp
# Open flamegraph.svg in browser
```

**samply (Cross-platform):**
```bash
cargo install samply
samply record target/release/myapp
```

### Memory Profiling

**DHAT (Heap profiling):**
```toml
[dependencies]
dhat = "0.3"
```

```rust
#[cfg(feature = "dhat")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat")]
    let _profiler = dhat::Profiler::new_heap();
    
    // Your code here
}
```

**Heaptrack (Linux):**
```bash
heaptrack target/release/myapp
heaptrack --analyze heaptrack.myapp.*.zst
```

### Finding Allocations

```rust
// Track allocations with a custom allocator
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering};

static ALLOCATED: AtomicUsize = AtomicUsize::new(0);

struct CountingAlloc;

unsafe impl GlobalAlloc for CountingAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOCATED.fetch_add(layout.size(), Ordering::SeqCst);
        System.alloc(layout)
    }
    
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        ALLOCATED.fetch_sub(layout.size(), Ordering::SeqCst);
        System.dealloc(ptr, layout)
    }
}

#[global_allocator]
static A: CountingAlloc = CountingAlloc;

fn main() {
    let before = ALLOCATED.load(Ordering::SeqCst);
    // Code to measure
    let after = ALLOCATED.load(Ordering::SeqCst);
    println!("Allocated {} bytes", after - before);
}
```

## Common Optimizations

### String Operations

```rust
// Slow: Many allocations
let mut s = String::new();
for i in 0..100 {
    s = s + &i.to_string();  // New allocation each time
}

// Better: Single buffer
let mut s = String::with_capacity(300);
for i in 0..100 {
    use std::fmt::Write;
    write!(s, "{}", i).unwrap();
}

// Best: Join iterator
let s: String = (0..100).map(|i| i.to_string()).collect();
```

### Hash Maps

```rust
use std::collections::HashMap;

// Pre-size when you know the count
let mut map = HashMap::with_capacity(1000);

// Use entry API (single lookup)
*map.entry(key).or_insert(0) += 1;

// Consider FxHashMap for integer keys
use rustc_hash::FxHashMap;
let mut map: FxHashMap<u64, Value> = FxHashMap::default();
```

### Iterators vs Loops

```rust
// Iterators often optimize better
let sum: i32 = data.iter().map(|x| x * 2).filter(|x| *x > 10).sum();

// But sometimes explicit loops are clearer and equally fast
let mut sum = 0;
for x in &data {
    let doubled = x * 2;
    if doubled > 10 {
        sum += doubled;
    }
}
```

### Avoid Bounds Checking

```rust
// Bounds check on each access
for i in 0..v.len() {
    process(v[i]);
}

// No bounds checks (iterator)
for item in &v {
    process(*item);
}

// Unsafe: skip bounds check (only when proven safe!)
unsafe {
    for i in 0..v.len() {
        process(*v.get_unchecked(i));
    }
}
```

### Stack vs Heap

```rust
// Heap allocated (Box)
let data = Box::new([0u8; 1024]);

// Stack allocated (if small enough)
let data = [0u8; 1024];

// Use arrays for fixed-size, known at compile time
// Use Vec for dynamic size
```

## Async Performance

```rust
// Avoid holding locks across awaits
let data = {
    let guard = mutex.lock().await;
    guard.clone()  // Clone and release lock
}; // Lock released here
process(data).await;  // Await without lock

// Use tokio::spawn for CPU-bound work
let result = tokio::task::spawn_blocking(|| {
    expensive_computation()
}).await?;

// Buffer I/O
use tokio::io::BufReader;
let reader = BufReader::new(file);
```

## Data Structure Choice

| Use Case | Data Structure | Why |
|----------|----------------|-----|
| Sequential access | `Vec<T>` | Cache-friendly |
| Key-value lookup | `HashMap` / `FxHashMap` | O(1) average |
| Sorted + lookup | `BTreeMap` | O(log n), ordered |
| Unique elements | `HashSet` | O(1) contains |
| FIFO queue | `VecDeque` | O(1) push/pop both ends |
| Small fixed set | `ArrayVec` / `SmallVec` | No heap allocation |
| Bit flags | `bitflags` | Memory efficient |

## Compiler Hints

```rust
// Likely/unlikely branches (nightly)
#![feature(core_intrinsics)]
use std::intrinsics::{likely, unlikely};

if unlikely(error_condition) {
    handle_error();
}

// Inline hints
#[inline]           // Suggest inlining
#[inline(always)]   // Force inlining
#[inline(never)]    // Prevent inlining
#[cold]             // Rarely called (optimize for space)
fn rarely_used() { ... }

// Target-specific optimization
#[cfg(target_feature = "avx2")]
fn simd_process(data: &[f32]) { ... }
```

## Arena Allocators

For batch allocations freed together (parsers, request processing):

```rust
use bumpalo::Bump;

// All allocations from one arena — freed together when arena drops
fn parse<'a>(input: &str, arena: &'a Bump) -> Vec<&'a Node> {
    tokenize(input).map(|t| arena.alloc(Node::new(t))).collect()
}

// Per-request: allocate freely, free everything at once
async fn handle(req: Request) -> Response {
    let arena = Bump::new();
    let parsed = parse(&req.body, &arena);
    generate_response(parsed)
} // arena dropped: all parsed nodes freed in one dealloc
```

## SmallVec — Inline Small Collections

```rust
use smallvec::SmallVec;

// Up to 4 items stored inline (stack), spills to heap beyond that
let mut tags: SmallVec<[&str; 4]> = SmallVec::new();
tags.push("performance");
tags.push("rust");
// No heap allocation for ≤ 4 items
```

## Write! Over format!

```rust
use std::fmt::Write;

// Bad: allocates a new String every time
let s = format!("key={} val={}", key, val);

// Good: write into pre-allocated buffer, clear and reuse
let mut buf = String::with_capacity(128);
for (key, val) in &map {
    buf.clear();
    write!(buf, "key={key} val={val}").unwrap();
    send(&buf);
}
```

## Entry API for HashMap

```rust
// Bad: two lookups
if let Some(v) = map.get_mut(&k) { *v += 1; } else { map.insert(k, 1); }

// Good: single lookup
*map.entry(k).or_insert(0) += 1;
```

## Struct Size — Keep It Small

```rust
use std::mem::size_of;

// Catch size regressions at compile time
const _: () = assert!(size_of::<MyEvent>() <= 64);

// Large enum variant → box it
enum Message {
    Ping,
    Text(String),
    // HugeThing(VeryLargeStruct), // Makes ALL variants huge!
    HugeThing(Box<VeryLargeStruct>), // Pointer-sized, heap only when needed
}
```

## Quick Checklist

- [ ] Profiled to find actual bottleneck?
- [ ] Running in release mode?
- [ ] Using `&str` instead of `String` where possible?
- [ ] Pre-allocating collections?
- [ ] Avoiding unnecessary clones?
- [ ] Using iterators over index loops?
- [ ] Using appropriate data structures?
- [ ] Benchmarked before and after?
- [ ] No intermediate `.collect()` in hot paths?
- [ ] Using entry API for HashMap inserts?
- [ ] Considered SmallVec or arena for short-lived allocations?

## References

- [The Rust Performance Book](https://nnethercote.github.io/perf-book/)
- [bumpalo](https://docs.rs/bumpalo) — arena allocator
- [smallvec](https://docs.rs/smallvec)
- [cargo-flamegraph](https://github.com/flamegraph-rs/flamegraph)
- [samply](https://github.com/mstange/samply) — cross-platform profiler
