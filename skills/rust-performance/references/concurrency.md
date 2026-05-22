# Concurrent Performance in Rust

Optimizing parallel and async code for maximum throughput.

## Parallelism with Rayon

### Basic Parallel Iteration

```rust
use rayon::prelude::*;

// Sequential
let sum: i64 = data.iter().map(|x| x * 2).sum();

// Parallel - automatic work stealing
let sum: i64 = data.par_iter().map(|x| x * 2).sum();

// Parallel collect
let results: Vec<_> = data.par_iter()
    .filter(|x| x.is_valid())
    .map(|x| process(x))
    .collect();
```

### When to Use Rayon

```rust
// GOOD: CPU-bound, large dataset
let hashes: Vec<_> = files.par_iter()
    .map(|f| compute_hash(f))
    .collect();

// GOOD: Independent computations
let results: Vec<_> = inputs.par_iter()
    .map(|input| expensive_computation(input))
    .collect();

// BAD: Small dataset (overhead > benefit)
let small: Vec<_> = [1, 2, 3].par_iter()
    .map(|x| x + 1)
    .collect();

// BAD: I/O bound (use async instead)
let responses: Vec<_> = urls.par_iter()
    .map(|url| blocking_http_get(url))  // Don't do this
    .collect();
```

### Controlling Parallelism

```rust
use rayon::ThreadPoolBuilder;

// Global thread pool configuration
ThreadPoolBuilder::new()
    .num_threads(4)
    .build_global()
    .unwrap();

// Local thread pool
let pool = ThreadPoolBuilder::new()
    .num_threads(2)
    .build()
    .unwrap();

pool.install(|| {
    // Parallel work uses this pool
    data.par_iter().for_each(|x| process(x));
});
```

### Parallel Chunks

```rust
// Process in parallel batches
data.par_chunks(1000)
    .for_each(|chunk| {
        process_batch(chunk);
    });

// Mutable chunks
data.par_chunks_mut(1000)
    .for_each(|chunk| {
        for item in chunk {
            *item = transform(*item);
        }
    });
```

## Lock-Free Data Structures

### Atomic Operations

```rust
use std::sync::atomic::{AtomicUsize, AtomicBool, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);
static FLAG: AtomicBool = AtomicBool::new(false);

// Increment
COUNTER.fetch_add(1, Ordering::Relaxed);

// Compare and swap
let old = COUNTER.compare_exchange(
    expected,
    new_value,
    Ordering::SeqCst,
    Ordering::Relaxed,
);

// Load/store
let value = COUNTER.load(Ordering::Acquire);
FLAG.store(true, Ordering::Release);
```

### Ordering Choices

| Ordering | Use Case | Performance |
|----------|----------|-------------|
| `Relaxed` | Counters, no sync needed | Fastest |
| `Acquire` | Load before critical section | Fast |
| `Release` | Store after critical section | Fast |
| `AcqRel` | Read-modify-write | Medium |
| `SeqCst` | Full ordering guarantee | Slowest |

```rust
// Counter - no ordering needed
static HITS: AtomicUsize = AtomicUsize::new(0);
HITS.fetch_add(1, Ordering::Relaxed);

// Flag to signal completion
static READY: AtomicBool = AtomicBool::new(false);
// Writer
data.store(value);
READY.store(true, Ordering::Release);
// Reader
if READY.load(Ordering::Acquire) {
    // data is guaranteed visible
}
```

### Crossbeam Channels

```rust
use crossbeam_channel::{bounded, unbounded, select};

// Bounded channel (backpressure)
let (tx, rx) = bounded::<Task>(100);

// Unbounded (no backpressure)
let (tx, rx) = unbounded::<Task>();

// Multiple producers
let tx2 = tx.clone();
std::thread::spawn(move || tx.send(task1));
std::thread::spawn(move || tx2.send(task2));

// Select from multiple channels
select! {
    recv(rx1) -> msg => handle1(msg),
    recv(rx2) -> msg => handle2(msg),
    send(tx, value) -> res => handle_send(res),
    default(Duration::from_millis(100)) => handle_timeout(),
}
```

## Lock Optimization

### RwLock for Read-Heavy Workloads

```rust
use std::sync::RwLock;

let data = RwLock::new(HashMap::new());

// Many readers can access simultaneously
let guard = data.read().unwrap();
let value = guard.get(&key);

// Writers get exclusive access
let mut guard = data.write().unwrap();
guard.insert(key, value);
```

### Sharded Locks

```rust
use dashmap::DashMap;

// Automatically sharded concurrent hashmap
let map: DashMap<String, Value> = DashMap::new();

// Multiple threads can access different shards simultaneously
map.insert("key".into(), value);
if let Some(v) = map.get("key") {
    println!("{:?}", *v);
}

// Entry API
map.entry("key".into())
    .or_insert_with(|| compute_value());
```

### Minimize Lock Scope

```rust
// BAD: Long lock hold
let mut guard = data.lock().unwrap();
let result = expensive_computation(&guard);
guard.update(result);

// GOOD: Short lock hold
let snapshot = {
    let guard = data.lock().unwrap();
    guard.clone()  // Clone and release
};
let result = expensive_computation(&snapshot);  // No lock held
{
    let mut guard = data.lock().unwrap();
    guard.update(result);  // Short lock
}
```

### Parking Lot

Faster synchronization primitives:

```rust
use parking_lot::{Mutex, RwLock, Condvar};

// Faster mutex (no poisoning)
let mutex = Mutex::new(0);
*mutex.lock() += 1;  // No unwrap needed

// Faster RwLock with upgradeable reads
let lock = RwLock::new(HashMap::new());
let read = lock.read();
if !read.contains_key(&key) {
    drop(read);
    let mut write = lock.write();
    write.insert(key, value);
}
```

## Async Performance

### Avoid Blocking in Async

```rust
// BAD: Blocks async runtime
async fn bad() {
    std::thread::sleep(Duration::from_secs(1));  // Blocks!
    std::fs::read_to_string("file.txt")?;         // Blocks!
}

// GOOD: Use async equivalents
async fn good() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    tokio::fs::read_to_string("file.txt").await?;
}

// GOOD: Offload blocking work
async fn good_blocking() {
    let result = tokio::task::spawn_blocking(|| {
        expensive_sync_operation()
    }).await?;
}
```

### Buffered I/O

```rust
use tokio::io::{BufReader, BufWriter, AsyncBufReadExt, AsyncWriteExt};

// Buffered reading
let file = tokio::fs::File::open("data.txt").await?;
let reader = BufReader::new(file);
let mut lines = reader.lines();
while let Some(line) = lines.next_line().await? {
    process(line);
}

// Buffered writing
let file = tokio::fs::File::create("output.txt").await?;
let mut writer = BufWriter::new(file);
for item in items {
    writer.write_all(item.as_bytes()).await?;
}
writer.flush().await?;
```

### Connection Pooling

```rust
use deadpool_postgres::{Config, Pool};

// Create pool
let pool: Pool = config.create_pool()?;

// Get connection from pool
async fn query(pool: &Pool) -> Result<Data> {
    let client = pool.get().await?;
    let rows = client.query("SELECT * FROM users", &[]).await?;
    Ok(rows)
}  // Connection returned to pool
```

### Batch Operations

```rust
// BAD: Sequential requests
for url in urls {
    let response = client.get(url).send().await?;
    results.push(response);
}

// GOOD: Concurrent requests
use futures::future::join_all;

let futures: Vec<_> = urls.iter()
    .map(|url| client.get(url).send())
    .collect();
let results = join_all(futures).await;

// GOOD: Limited concurrency
use futures::stream::{self, StreamExt};

let results: Vec<_> = stream::iter(urls)
    .map(|url| client.get(url).send())
    .buffer_unordered(10)  // Max 10 concurrent
    .collect()
    .await;
```

## Memory Ordering Patterns

### Producer-Consumer

```rust
use std::sync::atomic::{AtomicUsize, AtomicBool, Ordering};

static DATA: AtomicUsize = AtomicUsize::new(0);
static READY: AtomicBool = AtomicBool::new(false);

// Producer
DATA.store(42, Ordering::Relaxed);
READY.store(true, Ordering::Release);  // Release barrier

// Consumer
while !READY.load(Ordering::Acquire) {  // Acquire barrier
    std::hint::spin_loop();
}
let value = DATA.load(Ordering::Relaxed);  // Guaranteed to see 42
```

### Double-Checked Locking

```rust
use std::sync::{Once, OnceLock};

// OnceLock (recommended)
static CONFIG: OnceLock<Config> = OnceLock::new();

fn get_config() -> &'static Config {
    CONFIG.get_or_init(|| {
        load_config()  // Called exactly once
    })
}

// Once for side effects
static INIT: Once = Once::new();

fn ensure_initialized() {
    INIT.call_once(|| {
        initialize_subsystem();
    });
}
```

## Quick Reference

| Pattern | Use Case |
|---------|----------|
| Rayon | CPU-bound parallel work |
| Atomics | Simple counters/flags |
| DashMap | Concurrent hashmap |
| parking_lot | Faster locks |
| crossbeam | Lock-free channels |
| spawn_blocking | Blocking in async |
| buffer_unordered | Limited async concurrency |
| OnceLock | Lazy static initialization |
