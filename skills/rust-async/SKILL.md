---
name: rust-async
description: |
  Async Rust programming with Tokio, futures, channels, and concurrency patterns. ALWAYS use this skill when:
  - User writes async fn, uses .await, or works with Future trait
  - User uses Tokio runtime, tokio::spawn, spawn_blocking, or #[tokio::main]
  - User asks about concurrent requests, parallel tasks, or async I/O
  - User works with channels (mpsc, oneshot, broadcast, watch)
  - User asks about select!, join!, tokio::select, or futures::join
  - User has async deadlocks, "future is not Send", or runtime panics
  - User wants graceful shutdown, timeouts, or rate limiting in async code
  - User asks about Mutex, RwLock, Semaphore in async context
  - User builds web servers, scrapers, or networked applications in Rust
  - User mentions "concurrent", "parallel", or "non-blocking" with Rust
  Trigger for ANY async Rust code or Tokio-related questions.
---

# Async Rust Programming

Comprehensive guide to async programming with Tokio, futures, and concurrent patterns.

## Quick Navigation
- **references/tokio.md** - Tokio runtime deep dive
- **references/patterns.md** - Async patterns and anti-patterns
- **references/channels.md** - Channel types and message passing

## Core Concepts

### Async/Await Basics

```rust
// Async function returns a Future
async fn fetch_data(url: &str) -> Result<String, Error> {
    let response = reqwest::get(url).await?;
    let body = response.text().await?;
    Ok(body)
}

// Futures are lazy - nothing happens until awaited
let future = fetch_data("https://api.example.com");
// ... future not executed yet ...
let result = future.await; // Now it runs
```

### The Tokio Runtime

```rust
// Main entry point
#[tokio::main]
async fn main() {
    println!("Hello, async world!");
}

// Equivalent to:
fn main() {
    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async {
            println!("Hello, async world!");
        });
}

// Multi-threaded runtime (default)
#[tokio::main]
async fn main() { ... }

// Single-threaded runtime
#[tokio::main(flavor = "current_thread")]
async fn main() { ... }

// Custom runtime
#[tokio::main(worker_threads = 4)]
async fn main() { ... }
```

## Spawning Tasks

### tokio::spawn

Run tasks concurrently:

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    // Spawn concurrent tasks
    let handle1 = task::spawn(async {
        // Task 1 work
        "result1"
    });
    
    let handle2 = task::spawn(async {
        // Task 2 work  
        "result2"
    });
    
    // Wait for both
    let (result1, result2) = tokio::join!(handle1, handle2);
    println!("{:?}, {:?}", result1, result2);
}
```

### spawn_blocking

For CPU-bound work:

```rust
// Don't block async runtime with CPU work
let result = tokio::task::spawn_blocking(|| {
    // CPU-intensive computation
    expensive_hash_computation()
}).await?;

// Or for blocking I/O (non-async libraries)
let result = tokio::task::spawn_blocking(move || {
    std::fs::read_to_string("large_file.txt")
}).await??;
```

### Task Types

| Function | Use Case | Thread Pool |
|----------|----------|-------------|
| `spawn` | Async work | Async workers |
| `spawn_blocking` | CPU/blocking work | Blocking pool |
| `spawn_local` | Non-Send futures | Current thread |
| `block_in_place` | Blocking in async context | Current thread |

## Concurrency Primitives

### tokio::join! - Run Concurrently, Wait for All

```rust
use tokio::join;

async fn fetch_all() -> Result<(User, Posts, Comments), Error> {
    let (user, posts, comments) = join!(
        fetch_user(),
        fetch_posts(),
        fetch_comments()
    );
    
    Ok((user?, posts?, comments?))
}
```

### tokio::select! - First to Complete Wins

```rust
use tokio::{select, time::{sleep, Duration}};

async fn with_timeout() -> Result<Data, Error> {
    select! {
        result = fetch_data() => result,
        _ = sleep(Duration::from_secs(5)) => {
            Err(Error::Timeout)
        }
    }
}

// Multiple branches
select! {
    msg = rx.recv() => handle_message(msg),
    _ = shutdown_signal() => break,
    _ = interval.tick() => do_periodic_work(),
}
```

### FuturesUnordered - Dynamic Task Set

```rust
use futures::stream::{FuturesUnordered, StreamExt};

async fn process_urls(urls: Vec<String>) -> Vec<Result<String, Error>> {
    let mut futures = FuturesUnordered::new();
    
    for url in urls {
        futures.push(fetch(url));
    }
    
    let mut results = Vec::new();
    while let Some(result) = futures.next().await {
        results.push(result);
    }
    results
}
```

## Channels

### MPSC (Multi-Producer, Single-Consumer)

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(100);
    
    // Spawn producer
    let tx2 = tx.clone();
    tokio::spawn(async move {
        tx.send("Hello".to_string()).await.unwrap();
    });
    
    tokio::spawn(async move {
        tx2.send("World".to_string()).await.unwrap();
    });
    
    // Receive
    while let Some(msg) = rx.recv().await {
        println!("Got: {}", msg);
    }
}
```

### Oneshot (Single Value)

```rust
use tokio::sync::oneshot;

async fn request_response() {
    let (tx, rx) = oneshot::channel();
    
    tokio::spawn(async move {
        let result = compute().await;
        tx.send(result).unwrap();
    });
    
    let result = rx.await.unwrap();
}
```

### Broadcast (Multi-Consumer)

```rust
use tokio::sync::broadcast;

let (tx, _rx) = broadcast::channel::<Event>(100);

// Multiple receivers
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

tx.send(Event::Shutdown)?;

// Both receive the event
```

### Watch (Single Value, Multiple Observers)

```rust
use tokio::sync::watch;

let (tx, rx) = watch::channel(Config::default());

// Update value
tx.send(new_config)?;

// Observers receive latest value
let current = rx.borrow().clone();

// Wait for changes
rx.changed().await?;
```

### Channel Comparison

| Type | Producers | Consumers | Buffering |
|------|-----------|-----------|-----------|
| `mpsc` | Many | One | Bounded/Unbounded |
| `oneshot` | One | One | Single value |
| `broadcast` | One | Many | Bounded |
| `watch` | One | Many | Latest only |

## Synchronization

### Mutex

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

let data = Arc::new(Mutex::new(vec![]));

let data2 = data.clone();
tokio::spawn(async move {
    let mut guard = data2.lock().await;
    guard.push(1);
}); // Lock released when guard drops

// Don't hold locks across .await when possible
let item = {
    let guard = data.lock().await;
    guard.first().cloned()
}; // Lock released before await
process(item).await;
```

### RwLock

```rust
use tokio::sync::RwLock;

let data = RwLock::new(HashMap::new());

// Multiple readers
let reader1 = data.read().await;
let reader2 = data.read().await;

// Exclusive writer
let mut writer = data.write().await;
writer.insert("key", "value");
```

### Semaphore

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

let semaphore = Arc::new(Semaphore::new(10)); // Max 10 concurrent

async fn limited_task(sem: Arc<Semaphore>) {
    let permit = sem.acquire().await.unwrap();
    // Do work with permit held
    do_work().await;
    // Permit dropped, slot freed
}
```

## Async I/O

### File Operations

```rust
use tokio::fs;
use tokio::io::{AsyncReadExt, AsyncWriteExt, BufReader, BufWriter};

// Read file
let content = fs::read_to_string("file.txt").await?;

// Write file
fs::write("output.txt", content).await?;

// Buffered I/O
let file = fs::File::open("large.txt").await?;
let mut reader = BufReader::new(file);
let mut line = String::new();
reader.read_line(&mut line).await?;
```

### TCP

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

// Server
let listener = TcpListener::bind("127.0.0.1:8080").await?;
loop {
    let (socket, addr) = listener.accept().await?;
    tokio::spawn(async move {
        handle_connection(socket).await;
    });
}

// Client
let mut stream = TcpStream::connect("127.0.0.1:8080").await?;
stream.write_all(b"Hello").await?;
let mut buf = [0u8; 1024];
let n = stream.read(&mut buf).await?;
```

## Common Patterns

### Graceful Shutdown

```rust
use tokio::signal;
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (shutdown_tx, shutdown_rx) = watch::channel(false);
    
    let server = tokio::spawn(run_server(shutdown_rx.clone()));
    
    // Wait for Ctrl+C
    signal::ctrl_c().await.unwrap();
    
    // Signal shutdown
    shutdown_tx.send(true).unwrap();
    
    // Wait for server to finish
    server.await.unwrap();
}

async fn run_server(mut shutdown: watch::Receiver<bool>) {
    loop {
        select! {
            conn = accept_connection() => handle(conn).await,
            _ = shutdown.changed() => {
                if *shutdown.borrow() {
                    break;
                }
            }
        }
    }
}
```

### Rate Limiting

```rust
use tokio::time::{interval, Duration};

async fn rate_limited_work() {
    let mut interval = interval(Duration::from_millis(100));
    
    for item in items {
        interval.tick().await; // 10 per second max
        process(item).await;
    }
}
```

### Timeout

```rust
use tokio::time::{timeout, Duration};

let result = timeout(
    Duration::from_secs(5),
    fetch_data()
).await;

match result {
    Ok(data) => println!("Got data: {:?}", data),
    Err(_) => println!("Timed out"),
}
```

### Retry with Backoff

```rust
use tokio::time::{sleep, Duration};

async fn retry_with_backoff<F, Fut, T, E>(
    mut f: F,
    max_retries: u32,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    let mut delay = Duration::from_millis(100);
    
    for attempt in 0..max_retries {
        match f().await {
            Ok(result) => return Ok(result),
            Err(e) if attempt + 1 == max_retries => return Err(e),
            Err(_) => {
                sleep(delay).await;
                delay *= 2; // Exponential backoff
            }
        }
    }
    unreachable!()
}
```

## Anti-Patterns

### Holding Locks Across Await

```rust
// BAD: Lock held across await
let guard = mutex.lock().await;
some_async_operation().await; // Lock still held!
drop(guard);

// GOOD: Clone and release
let data = {
    let guard = mutex.lock().await;
    guard.clone()
};
some_async_operation_with(data).await;
```

### Blocking in Async Context

```rust
// BAD: Blocks the async runtime
async fn bad() {
    std::thread::sleep(Duration::from_secs(1)); // Blocks!
    std::fs::read_to_string("file.txt"); // Blocks!
}

// GOOD: Use async equivalents
async fn good() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    tokio::fs::read_to_string("file.txt").await;
}

// Or spawn_blocking for unavoidable blocking
let result = tokio::task::spawn_blocking(|| {
    blocking_library_call()
}).await?;
```

### Creating Runtime in Async Context

```rust
// BAD: Nested runtime
async fn bad() {
    tokio::runtime::Runtime::new().unwrap()
        .block_on(async { ... }); // Panic or deadlock!
}

// GOOD: Just await
async fn good() {
    some_future().await;
}
```

## Debugging Tips

1. **Use `#[tokio::test]`** for async tests
2. **Enable tracing** with `tracing` crate
3. **Use `tokio-console`** for runtime inspection
4. **Check for dropped futures** (no `.await`)
5. **Look for blocking calls** in async code
