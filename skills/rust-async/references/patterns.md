# Async Patterns and Anti-Patterns

Common patterns for writing correct and efficient async Rust code.

## Concurrency Patterns

### Fan-Out / Fan-In

Process items concurrently, collect results:

```rust
use futures::future::join_all;

async fn process_all(items: Vec<Item>) -> Vec<Result<Output, Error>> {
    let futures: Vec<_> = items.into_iter()
        .map(|item| process_item(item))
        .collect();
    
    join_all(futures).await
}
```

### Bounded Concurrency

Limit concurrent operations:

```rust
use futures::stream::{self, StreamExt};

async fn process_bounded(items: Vec<Item>, concurrency: usize) -> Vec<Output> {
    stream::iter(items)
        .map(|item| async move {
            process_item(item).await
        })
        .buffer_unordered(concurrency)  // Max concurrent
        .collect()
        .await
}
```

### First Success

Return first successful result:

```rust
use futures::future::select_all;

async fn first_success<T, E>(
    futures: Vec<impl Future<Output = Result<T, E>>>
) -> Result<T, E> {
    let mut futures = futures;
    
    while !futures.is_empty() {
        let (result, _index, remaining) = select_all(futures).await;
        
        if result.is_ok() {
            return result;
        }
        
        futures = remaining;
    }
    
    Err(/* all failed */)
}
```

### Racing with Timeout

```rust
use tokio::time::{timeout, Duration};

async fn with_timeout<T>(
    fut: impl Future<Output = T>,
    dur: Duration,
) -> Option<T> {
    timeout(dur, fut).await.ok()
}

// Multiple races
async fn fastest_response(urls: &[&str]) -> Option<Response> {
    let futures: Vec<_> = urls.iter()
        .map(|url| fetch(url))
        .collect();
    
    let (result, _, _) = futures::future::select_all(futures).await;
    result.ok()
}
```

## Cancellation Patterns

### Graceful Shutdown

```rust
use tokio::sync::watch;
use tokio::select;

struct Service {
    shutdown_tx: watch::Sender<bool>,
    shutdown_rx: watch::Receiver<bool>,
}

impl Service {
    fn new() -> Self {
        let (shutdown_tx, shutdown_rx) = watch::channel(false);
        Self { shutdown_tx, shutdown_rx }
    }
    
    async fn run(&self) {
        let mut shutdown = self.shutdown_rx.clone();
        
        loop {
            select! {
                _ = shutdown.changed() => {
                    if *shutdown.borrow() {
                        println!("Shutting down...");
                        break;
                    }
                }
                result = self.do_work() => {
                    // Handle work result
                }
            }
        }
    }
    
    fn shutdown(&self) {
        let _ = self.shutdown_tx.send(true);
    }
}
```

### CancellationToken

```rust
use tokio_util::sync::CancellationToken;

async fn cancellable_work(token: CancellationToken) {
    loop {
        select! {
            _ = token.cancelled() => {
                println!("Cancelled!");
                break;
            }
            result = do_work() => {
                // Process result
            }
        }
    }
}

// Usage
let token = CancellationToken::new();
let child_token = token.child_token();

tokio::spawn(cancellable_work(child_token));

// Later...
token.cancel();  // Cancels all child tokens
```

### Drop Guard

```rust
struct DropGuard<F: FnOnce()> {
    f: Option<F>,
}

impl<F: FnOnce()> Drop for DropGuard<F> {
    fn drop(&mut self) {
        if let Some(f) = self.f.take() {
            f();
        }
    }
}

async fn with_cleanup() {
    let _guard = DropGuard {
        f: Some(|| println!("Cleaning up!")),
    };
    
    // If future is dropped/cancelled, cleanup still runs
    do_work().await;
}
```

## Error Handling Patterns

### Try-Join

Stop on first error:

```rust
use tokio::try_join;

async fn fetch_all() -> Result<(User, Posts), Error> {
    let (user, posts) = try_join!(
        fetch_user(),
        fetch_posts()
    )?;  // Returns early if either fails
    
    Ok((user, posts))
}
```

### Collect Results

Continue despite errors:

```rust
async fn process_all(items: Vec<Item>) -> (Vec<Output>, Vec<Error>) {
    let results = futures::future::join_all(
        items.into_iter().map(process_item)
    ).await;
    
    let mut successes = Vec::new();
    let mut failures = Vec::new();
    
    for result in results {
        match result {
            Ok(output) => successes.push(output),
            Err(e) => failures.push(e),
        }
    }
    
    (successes, failures)
}
```

### Retry Pattern

```rust
use tokio::time::{sleep, Duration};
use std::future::Future;

async fn retry<T, E, F, Fut>(
    mut f: F,
    max_attempts: u32,
    initial_delay: Duration,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    let mut delay = initial_delay;
    
    for attempt in 1..=max_attempts {
        match f().await {
            Ok(result) => return Ok(result),
            Err(e) if attempt == max_attempts => return Err(e),
            Err(_) => {
                sleep(delay).await;
                delay *= 2;  // Exponential backoff
            }
        }
    }
    
    unreachable!()
}

// Usage
let result = retry(
    || fetch_data(url),
    3,
    Duration::from_millis(100),
).await?;
```

## Stream Patterns

### Async Iterator

```rust
use futures::stream::{self, StreamExt};

async fn process_stream() {
    let stream = stream::iter(1..=10)
        .map(|x| async move {
            tokio::time::sleep(Duration::from_millis(100)).await;
            x * 2
        })
        .buffer_unordered(3);  // Process 3 at a time
    
    pin_mut!(stream);
    
    while let Some(value) = stream.next().await {
        println!("{}", value);
    }
}
```

### Chunked Processing

```rust
use futures::stream::StreamExt;

async fn process_in_batches<T>(
    items: impl Stream<Item = T>,
    batch_size: usize,
) {
    let mut items = items.chunks(batch_size);
    
    while let Some(batch) = items.next().await {
        process_batch(batch).await;
    }
}
```

## Anti-Patterns

### Blocking in Async

```rust
// BAD: Blocks async thread
async fn bad() {
    std::thread::sleep(Duration::from_secs(1));  // BLOCKS!
    std::fs::read_to_string("file.txt")?;         // BLOCKS!
}

// GOOD: Use async equivalents
async fn good() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    tokio::fs::read_to_string("file.txt").await?;
}

// GOOD: Spawn blocking
async fn good_blocking() {
    tokio::task::spawn_blocking(|| {
        std::thread::sleep(Duration::from_secs(1));
    }).await?;
}
```

### Holding Locks Across Await

```rust
// BAD: Lock held during await
async fn bad(data: &Mutex<Data>) {
    let mut guard = data.lock().await;
    fetch_remote().await;  // Lock still held!
    guard.update();
}

// GOOD: Release lock before await
async fn good(data: &Mutex<Data>) {
    let snapshot = {
        let guard = data.lock().await;
        guard.clone()
    };  // Lock released
    
    let result = process(snapshot).await;
    
    let mut guard = data.lock().await;
    guard.apply(result);
}
```

### Unbounded Concurrency

```rust
// BAD: May spawn millions of tasks
async fn bad(items: Vec<Item>) {
    for item in items {
        tokio::spawn(async move {
            process(item).await;
        });
    }
}

// GOOD: Bounded concurrency
async fn good(items: Vec<Item>) {
    stream::iter(items)
        .map(|item| async move { process(item).await })
        .buffer_unordered(100)  // Max 100 concurrent
        .collect::<Vec<_>>()
        .await;
}
```

### Forgetting to Await

```rust
// BAD: Future never runs!
async fn bad() {
    async_operation();  // Returns Future, never awaited!
}

// GOOD: Await the future
async fn good() {
    async_operation().await;
}

// GOOD: Spawn if you don't need result
async fn good_spawn() {
    tokio::spawn(async_operation());
}
```

### Creating Runtime in Async

```rust
// BAD: Panics or deadlocks
async fn bad() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async { ... });  // BAD!
}

// GOOD: Just await
async fn good() {
    some_future().await;
}
```

## Performance Patterns

### Batch Database Operations

```rust
async fn insert_users(users: Vec<User>) -> Result<(), Error> {
    // BAD: One query per user
    for user in &users {
        db.insert_one(user).await?;
    }
    
    // GOOD: Batch insert
    db.insert_many(&users).await?;
    Ok(())
}
```

### Connection Reuse

```rust
// BAD: New connection per request
async fn bad(url: &str) -> Result<Response, Error> {
    let client = reqwest::Client::new();
    client.get(url).send().await
}

// GOOD: Reuse client
struct Service {
    client: reqwest::Client,  // Connection pooling
}

impl Service {
    async fn fetch(&self, url: &str) -> Result<Response, Error> {
        self.client.get(url).send().await
    }
}
```

### Avoid Unnecessary Boxing

```rust
// Adds allocation
async fn boxed() -> Box<dyn Future<Output = i32>> {
    Box::new(async { 42 })
}

// Zero-cost
async fn not_boxed() -> i32 {
    42
}

// Use impl Trait when possible
fn returns_future() -> impl Future<Output = i32> {
    async { 42 }
}
```
