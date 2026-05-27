# Async Channels in Depth

Complete guide to message passing in async Rust.

## Channel Types Overview

| Channel | Producers | Consumers | Buffer | Use Case |
|---------|-----------|-----------|--------|----------|
| `mpsc` | Many | One | Bounded/Unbounded | Task coordination |
| `oneshot` | One | One | Single value | Request-response |
| `broadcast` | One | Many | Bounded | Pub/sub, events |
| `watch` | One | Many | Latest only | Config, state |

## MPSC (Multi-Producer, Single-Consumer)

### Bounded Channel

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // Create channel with buffer of 100
    let (tx, mut rx) = mpsc::channel::<Message>(100);
    
    // Clone sender for multiple producers
    let tx2 = tx.clone();
    
    // Producer 1
    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(Message::Data(i)).await.unwrap();
        }
    });
    
    // Producer 2
    tokio::spawn(async move {
        tx2.send(Message::Control("start")).await.unwrap();
    });
    
    // Consumer
    while let Some(msg) = rx.recv().await {
        match msg {
            Message::Data(n) => println!("Got data: {}", n),
            Message::Control(cmd) => println!("Got command: {}", cmd),
        }
    }
}
```

### Backpressure

```rust
// Bounded channel applies backpressure
let (tx, mut rx) = mpsc::channel::<i32>(10);

tokio::spawn(async move {
    for i in 0..1000 {
        // Blocks when buffer full
        tx.send(i).await.unwrap();
        println!("Sent {}", i);
    }
});

// Slow consumer
while let Some(n) = rx.recv().await {
    tokio::time::sleep(Duration::from_millis(100)).await;
    println!("Received {}", n);
}
```

### Unbounded Channel

```rust
use tokio::sync::mpsc::unbounded_channel;

let (tx, mut rx) = unbounded_channel::<i32>();

// Never blocks on send
tx.send(42).unwrap();  // Note: not async!

// Use carefully - can cause memory issues
```

### Send Patterns

```rust
// send - blocks if full
tx.send(value).await?;

// try_send - returns immediately
match tx.try_send(value) {
    Ok(()) => println!("Sent!"),
    Err(TrySendError::Full(value)) => println!("Buffer full"),
    Err(TrySendError::Closed(value)) => println!("Receiver dropped"),
}

// send_timeout
tx.send_timeout(value, Duration::from_secs(1)).await?;

// Check capacity
if tx.capacity() > 0 {
    tx.send(value).await?;
}
```

### Receiver Patterns

```rust
// recv - blocks until message or closed
match rx.recv().await {
    Some(msg) => handle(msg),
    None => println!("Channel closed"),
}

// try_recv - non-blocking
match rx.try_recv() {
    Ok(msg) => handle(msg),
    Err(TryRecvError::Empty) => println!("No message"),
    Err(TryRecvError::Disconnected) => println!("Closed"),
}

// Batch receive
let mut batch = Vec::new();
while let Ok(msg) = rx.try_recv() {
    batch.push(msg);
    if batch.len() >= 100 { break; }
}
```

## Oneshot (Single Value)

### Oneshot Basics

```rust
use tokio::sync::oneshot;

async fn request_response() {
    let (tx, rx) = oneshot::channel();
    
    // Spawn task that sends response
    tokio::spawn(async move {
        let result = compute_something().await;
        tx.send(result).unwrap();
    });
    
    // Wait for response
    let response = rx.await.unwrap();
}
```

### Request-Response Pattern

```rust
use tokio::sync::{mpsc, oneshot};

enum Request {
    Get { key: String, respond_to: oneshot::Sender<Option<Value>> },
    Set { key: String, value: Value, respond_to: oneshot::Sender<bool> },
}

struct Actor {
    receiver: mpsc::Receiver<Request>,
    data: HashMap<String, Value>,
}

impl Actor {
    async fn run(&mut self) {
        while let Some(request) = self.receiver.recv().await {
            match request {
                Request::Get { key, respond_to } => {
                    let value = self.data.get(&key).cloned();
                    let _ = respond_to.send(value);
                }
                Request::Set { key, value, respond_to } => {
                    self.data.insert(key, value);
                    let _ = respond_to.send(true);
                }
            }
        }
    }
}

// Client
async fn get_value(sender: &mpsc::Sender<Request>, key: &str) -> Option<Value> {
    let (tx, rx) = oneshot::channel();
    sender.send(Request::Get {
        key: key.to_string(),
        respond_to: tx,
    }).await.ok()?;
    rx.await.ok()?
}
```

### Timeout with Oneshot

```rust
use tokio::time::timeout;

let (tx, rx) = oneshot::channel();

// Spawn worker
tokio::spawn(async move {
    let result = slow_operation().await;
    let _ = tx.send(result);
});

// Wait with timeout
match timeout(Duration::from_secs(5), rx).await {
    Ok(Ok(result)) => println!("Got: {:?}", result),
    Ok(Err(_)) => println!("Sender dropped"),
    Err(_) => println!("Timeout"),
}
```

## Broadcast (Multi-Consumer)

### Broadcast Basics

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, _rx) = broadcast::channel::<Event>(100);
    
    // Create subscribers
    let mut rx1 = tx.subscribe();
    let mut rx2 = tx.subscribe();
    
    // Publisher
    tokio::spawn(async move {
        tx.send(Event::Update("data")).unwrap();
    });
    
    // Subscribers receive same message
    let event1 = rx1.recv().await.unwrap();
    let event2 = rx2.recv().await.unwrap();
}
```

### Handling Lag

```rust
use tokio::sync::broadcast::{self, error::RecvError};

let (tx, mut rx) = broadcast::channel(100);

loop {
    match rx.recv().await {
        Ok(msg) => handle(msg),
        Err(RecvError::Lagged(count)) => {
            // Receiver too slow, missed `count` messages
            println!("Missed {} messages", count);
        }
        Err(RecvError::Closed) => break,
    }
}
```

### Event Bus Pattern

```rust
use tokio::sync::broadcast;

#[derive(Clone)]
enum Event {
    UserJoined(String),
    UserLeft(String),
    Message { from: String, text: String },
}

struct EventBus {
    sender: broadcast::Sender<Event>,
}

impl EventBus {
    fn new() -> Self {
        let (sender, _) = broadcast::channel(1000);
        Self { sender }
    }
    
    fn subscribe(&self) -> broadcast::Receiver<Event> {
        self.sender.subscribe()
    }
    
    fn publish(&self, event: Event) {
        let _ = self.sender.send(event);
    }
}
```

## Watch (Single Value, Latest Only)

### Basic Usage

```rust
use tokio::sync::watch;

let (tx, rx) = watch::channel(Config::default());

// Update value
tx.send(new_config)?;

// Read current value
let current = rx.borrow().clone();

// Wait for changes
rx.changed().await?;
let updated = rx.borrow().clone();
```

### Configuration Hot-Reload

```rust
use tokio::sync::watch;

struct ConfigWatcher {
    tx: watch::Sender<Config>,
    rx: watch::Receiver<Config>,
}

impl ConfigWatcher {
    fn new(initial: Config) -> Self {
        let (tx, rx) = watch::channel(initial);
        Self { tx, rx }
    }
    
    fn subscribe(&self) -> watch::Receiver<Config> {
        self.rx.clone()
    }
    
    async fn watch_file(&self, path: &str) {
        loop {
            tokio::time::sleep(Duration::from_secs(5)).await;
            
            if let Ok(config) = load_config(path).await {
                let _ = self.tx.send(config);
            }
        }
    }
}

// In worker
async fn worker(mut config_rx: watch::Receiver<Config>) {
    let mut config = config_rx.borrow().clone();
    
    loop {
        tokio::select! {
            _ = config_rx.changed() => {
                config = config_rx.borrow().clone();
                println!("Config updated!");
            }
            _ = do_work(&config) => {}
        }
    }
}
```

## Advanced Patterns

### Fan-Out with Broadcast

```rust
async fn fan_out<T: Clone + Send + 'static>(
    input: mpsc::Receiver<T>,
    outputs: Vec<mpsc::Sender<T>>,
) {
    let (broadcast_tx, _) = broadcast::channel(100);
    
    // Create receivers for each output
    let mut handles = Vec::new();
    for output in outputs {
        let mut rx = broadcast_tx.subscribe();
        handles.push(tokio::spawn(async move {
            while let Ok(item) = rx.recv().await {
                if output.send(item).await.is_err() {
                    break;
                }
            }
        }));
    }
    
    // Forward input to broadcast
    let mut input = input;
    while let Some(item) = input.recv().await {
        let _ = broadcast_tx.send(item);
    }
}
```

### Merge Channels

```rust
use futures::stream::{self, StreamExt};

async fn merge_channels(
    mut rx1: mpsc::Receiver<i32>,
    mut rx2: mpsc::Receiver<i32>,
) -> Vec<i32> {
    let mut results = Vec::new();
    
    loop {
        tokio::select! {
            Some(v) = rx1.recv() => results.push(v),
            Some(v) = rx2.recv() => results.push(v),
            else => break,
        }
    }
    
    results
}
```

### Priority Channel

```rust
use std::cmp::Ordering;
use std::collections::BinaryHeap;
use tokio::sync::{mpsc, Mutex};

struct PriorityItem<T> {
    priority: u32,
    item: T,
}

impl<T> Ord for PriorityItem<T> {
    fn cmp(&self, other: &Self) -> Ordering {
        self.priority.cmp(&other.priority)
    }
}

struct PriorityChannel<T> {
    sender: mpsc::Sender<PriorityItem<T>>,
    heap: Mutex<BinaryHeap<PriorityItem<T>>>,
}
```

## Performance Tips

### Buffer Sizing

```rust
// Too small: frequent backpressure
let (tx, rx) = mpsc::channel(1);

// Too large: memory waste
let (tx, rx) = mpsc::channel(1_000_000);

// Right size: based on throughput
// If producer sends 1000/sec and consumer processes 800/sec
// Buffer ~= burst_size + (producer_rate - consumer_rate) * acceptable_delay
let (tx, rx) = mpsc::channel(500);
```

### Batch Sending

```rust
// Send many small messages
for item in items {
    tx.send(item).await?;  // Overhead per send
}

// Better: batch messages
tx.send(items).await?;  // Single send of Vec
```

### Channel Selection

- **mpsc**: Default choice for task coordination
- **oneshot**: Single response, one-time use
- **broadcast**: Events to multiple subscribers
- **watch**: Shared state, only latest matters
- **crossbeam**: When you need sync channels
