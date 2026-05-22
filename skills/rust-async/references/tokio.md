# Tokio Runtime Deep Dive

Comprehensive guide to Tokio runtime configuration, features, and internals.

## Runtime Types

### Multi-Threaded Runtime

Default, production runtime with work-stealing scheduler:

```rust
#[tokio::main]
async fn main() {
    // Uses multi-threaded runtime
}

// Explicit configuration
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() {
    // 4 worker threads
}

// Manual construction
let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)
    .enable_all()
    .build()
    .unwrap();

rt.block_on(async {
    // Your async code
});
```

### Current-Thread Runtime

Single-threaded, lower overhead:

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {
    // Everything runs on main thread
}

// Manual
let rt = tokio::runtime::Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();
```

### Runtime Comparison

| Feature | Multi-Thread | Current-Thread |
|---------|-------------|----------------|
| Parallelism | Yes | No |
| Overhead | Higher | Lower |
| Use Case | CPU + I/O | Light I/O only |
| Send requirement | Yes | No |

## Runtime Configuration

### Thread Pool Settings

```rust
let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)           // Async worker threads
    .max_blocking_threads(512)   // For spawn_blocking
    .thread_name("my-worker")    // Thread names
    .thread_stack_size(3 * 1024 * 1024)  // Stack size
    .on_thread_start(|| {
        println!("Thread started");
    })
    .on_thread_stop(|| {
        println!("Thread stopped");
    })
    .build()
    .unwrap();
```

### Feature Selection

```rust
// Enable only what you need
let rt = tokio::runtime::Builder::new_current_thread()
    .enable_io()     // Async I/O
    .enable_time()   // Timers
    .build()
    .unwrap();

// Or enable all
let rt = tokio::runtime::Builder::new_multi_thread()
    .enable_all()
    .build()
    .unwrap();
```

## Task Spawning

### tokio::spawn

Spawns task on runtime's thread pool:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // Runs on any worker thread
        expensive_async_work().await
    });
    
    let result = handle.await.unwrap();
}
```

**Requirements:**
- Future must be `Send` (can move between threads)
- Future must be `'static` (no borrowed references)

```rust
// ERROR: Not Send
let rc = Rc::new(42);
tokio::spawn(async move {
    println!("{}", rc);  // Rc is not Send!
});

// ERROR: Not 'static
let data = String::from("hello");
tokio::spawn(async {
    println!("{}", data);  // Borrowed, not owned!
});

// OK: Move ownership
let data = String::from("hello");
tokio::spawn(async move {
    println!("{}", data);  // Moved, now 'static
});
```

### spawn_local

For non-Send futures (current-thread only):

```rust
use tokio::task::LocalSet;

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let local = LocalSet::new();
    
    local.run_until(async {
        tokio::task::spawn_local(async {
            // Can use non-Send types
            let rc = Rc::new(42);
            println!("{}", rc);
        }).await.unwrap();
    }).await;
}
```

### spawn_blocking

For CPU-bound or blocking operations:

```rust
// Don't block async workers with CPU work!
let result = tokio::task::spawn_blocking(|| {
    // Runs on dedicated blocking thread pool
    compute_hash(&data)
}).await?;

// For blocking I/O libraries
let content = tokio::task::spawn_blocking(move || {
    std::fs::read_to_string("file.txt")
}).await??;
```

### block_in_place

Block current thread while keeping runtime alive:

```rust
async fn mixed_work() {
    // Async work
    fetch_data().await;
    
    // Block without spawn overhead
    tokio::task::block_in_place(|| {
        expensive_sync_computation()
    });
    
    // More async work
    send_result().await;
}
```

## Task Local Storage

### task_local!

Per-task storage (like thread-local but for tasks):

```rust
tokio::task_local! {
    static REQUEST_ID: u64;
}

async fn handle_request(id: u64) {
    REQUEST_ID.scope(id, async {
        // REQUEST_ID is available here
        process().await;
    }).await;
}

async fn process() {
    let id = REQUEST_ID.get();
    println!("Processing request {}", id);
}
```

## Async I/O

### TCP

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

// Server
async fn run_server() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    
    loop {
        let (socket, addr) = listener.accept().await?;
        tokio::spawn(async move {
            handle_client(socket).await;
        });
    }
}

async fn handle_client(mut socket: TcpStream) {
    let mut buf = [0u8; 1024];
    
    loop {
        let n = socket.read(&mut buf).await.unwrap();
        if n == 0 { break; }
        socket.write_all(&buf[..n]).await.unwrap();
    }
}

// Client
async fn connect() -> std::io::Result<()> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;
    stream.write_all(b"hello").await?;
    
    let mut buf = [0u8; 1024];
    let n = stream.read(&mut buf).await?;
    println!("Received: {}", String::from_utf8_lossy(&buf[..n]));
    Ok(())
}
```

### File I/O

```rust
use tokio::fs::{self, File};
use tokio::io::{AsyncReadExt, AsyncWriteExt, BufReader, BufWriter};

// Simple read/write
let content = fs::read_to_string("file.txt").await?;
fs::write("output.txt", content).await?;

// Streaming read
let file = File::open("large.txt").await?;
let mut reader = BufReader::new(file);
let mut line = String::new();
reader.read_line(&mut line).await?;

// Streaming write
let file = File::create("output.txt").await?;
let mut writer = BufWriter::new(file);
writer.write_all(b"data").await?;
writer.flush().await?;
```

## Timers

### Sleep and Timeout

```rust
use tokio::time::{sleep, timeout, Duration};

// Sleep
sleep(Duration::from_secs(1)).await;

// Timeout
match timeout(Duration::from_secs(5), fetch_data()).await {
    Ok(result) => println!("Got: {:?}", result),
    Err(_) => println!("Timed out"),
}
```

### Interval

```rust
use tokio::time::{interval, Duration};

let mut interval = interval(Duration::from_secs(1));

loop {
    interval.tick().await;
    println!("Tick!");
}
```

### Delayed Execution

```rust
use tokio::time::{sleep_until, Instant, Duration};

// At specific instant
let deadline = Instant::now() + Duration::from_secs(5);
sleep_until(deadline).await;
```

## Signal Handling

```rust
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

## Runtime Metrics (Unstable)

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::runtime::Handle::current();
    let metrics = handle.metrics();
    
    println!("Workers: {}", metrics.num_workers());
    println!("Blocking threads: {}", metrics.num_blocking_threads());
    println!("Active tasks: {}", metrics.active_tasks_count());
}
```

## Best Practices

### Do's

```rust
// Use async equivalents
tokio::time::sleep(duration).await;
tokio::fs::read_to_string(path).await;

// Use spawn_blocking for CPU work
tokio::task::spawn_blocking(|| heavy_computation()).await;

// Use buffered I/O
let reader = BufReader::new(file);
let writer = BufWriter::new(file);

// Pre-size channels
let (tx, rx) = mpsc::channel(1000);
```

### Don'ts

```rust
// Don't block async threads
std::thread::sleep(duration);  // BAD
std::fs::read_to_string(path); // BAD

// Don't hold locks across .await
let guard = mutex.lock().await;
some_async_op().await;  // Lock held! BAD
drop(guard);

// Don't create runtime inside async
async fn bad() {
    let rt = Runtime::new();  // Panic or deadlock!
}
```
