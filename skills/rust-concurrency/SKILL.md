---
name: rust-concurrency
description: |
  Rust concurrency — threads, channels, shared state, Send/Sync. Use this skill when:
  - User asks about std::thread, thread::spawn, or thread::scope
  - User encounters E0277 "not Send" or "not Sync" errors
  - User uses Arc, Mutex, RwLock, or atomic types
  - User works with std::sync::mpsc or crossbeam channels
  - User implements a thread pool or work-stealing pattern
  - User asks about rayon for data parallelism
  - User encounters deadlocks or race conditions
  - User asks about lock-free data structures or atomics
  - User asks "when should I use threads vs async?"
  Trigger for OS thread concurrency questions (use rust-async for Tokio/async).
---

# Rust Concurrency

Rust's ownership model prevents data races at compile time. If it compiles, it's data-race-free.

## The Core Guarantee

```
Send: A type can be transferred to another thread
Sync: A type can be shared (via &T) across threads

// Compiler-enforced rules:
T: Send     → you can move T to another thread
T: Sync     → you can share &T across threads
Arc<T>: Send iff T: Send + Sync
Mutex<T>: Send iff T: Send
```

## Threads vs Async

| | `std::thread` | `tokio::spawn` |
|---|---|---|
| Best for | CPU-bound, blocking | I/O-bound, many connections |
| Cost | OS thread (~1MB stack) | Task (~few KB) |
| Blocking | Natural | Must use spawn_blocking |
| Parallelism | True parallel | Concurrent (1+ threads) |

**Use threads when**: crunching numbers, image processing, parsing large files.
**Use async when**: waiting on network, database, disk I/O.

## Basic Threads

```rust
use std::thread;
use std::time::Duration;

// Spawn a thread
let handle = thread::spawn(|| {
    println!("Hello from thread!");
    42
});

let result = handle.join().unwrap(); // Wait and get return value
println!("Thread returned: {result}");

// Move data into thread
let data = vec![1, 2, 3];
let handle = thread::spawn(move || {  // `move` captures data by value
    println!("Got: {:?}", data);
    data.len()
});
```

### Thread Scope (Borrow in Threads)

```rust
use std::thread;

let data = vec![1, 2, 3, 4, 5, 6, 7, 8];

// thread::scope lets you borrow data — threads can't outlive the scope
thread::scope(|s| {
    let chunk1 = &data[..4];
    let chunk2 = &data[4..];

    let t1 = s.spawn(|| chunk1.iter().sum::<i32>());
    let t2 = s.spawn(|| chunk2.iter().sum::<i32>());

    let sum = t1.join().unwrap() + t2.join().unwrap();
    println!("Sum: {sum}");
}); // Scope ensures all threads finish before data is dropped
```

## Shared State

### Arc<Mutex<T>>

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0u64));

let handles: Vec<_> = (0..10).map(|_| {
    let counter = Arc::clone(&counter); // Cheap: increments reference count
    thread::spawn(move || {
        let mut guard = counter.lock().unwrap();
        *guard += 1;
        // Guard dropped here: lock released
    })
}).collect();

for h in handles { h.join().unwrap(); }
println!("Final: {}", *counter.lock().unwrap()); // 10
```

**Pitfalls:**
```rust
// BAD: Holding lock across blocking operations
let guard = mutex.lock().unwrap();
some_slow_io_call(); // Other threads blocked the whole time!

// GOOD: Clone data, release lock early
let data = mutex.lock().unwrap().clone();
drop(data); // Explicitly release
some_slow_io_call();
```

### Arc<RwLock<T>> — Many Readers, One Writer

```rust
use std::sync::{Arc, RwLock};

let config = Arc::new(RwLock::new(Config::default()));

// Many readers can hold read lock simultaneously
let reader1 = {
    let c = Arc::clone(&config);
    thread::spawn(move || {
        let guard = c.read().unwrap();
        println!("timeout: {}", guard.timeout);
    })
};

// Writer blocks until all readers release
let writer = {
    let c = Arc::clone(&config);
    thread::spawn(move || {
        let mut guard = c.write().unwrap();
        guard.timeout = 60;
    })
};
```

**RwLock risk**: writer starvation. If readers are constant, writers wait forever.
For read-heavy workloads with infrequent writes, consider `arc-swap` crate.

## Channels (Message Passing)

### std::sync::mpsc

```rust
use std::sync::mpsc;
use std::thread;

// Unbounded channel
let (tx, rx) = mpsc::channel::<String>();

// Multiple senders
for i in 0..5 {
    let tx = tx.clone();
    thread::spawn(move || {
        tx.send(format!("Message {i}")).unwrap();
    });
}
drop(tx); // Close original sender so rx knows when all senders are done

// Receive all messages
for msg in rx {
    println!("Got: {msg}");
}

// Bounded (backpressure)
let (tx, rx) = mpsc::sync_channel::<u32>(16);
```

### Crossbeam Channels (Better)

```rust
use crossbeam::channel::{bounded, unbounded, select};

// Multiple producers AND consumers
let (tx, rx) = bounded::<Work>(100);

// Producer threads
for i in 0..4 {
    let tx = tx.clone();
    thread::spawn(move || {
        loop {
            tx.send(Work::new(i)).unwrap();
        }
    });
}

// Consumer threads
for _ in 0..4 {
    let rx = rx.clone(); // crossbeam channels are multi-consumer
    thread::spawn(move || {
        for work in &rx {
            process(work);
        }
    });
}

// select! — first-ready wins
select! {
    recv(rx1) -> msg => handle_msg(msg),
    recv(rx2) -> msg => handle_msg(msg),
    recv(timeout_rx) -> _ => return Err(Timeout),
}
```

## Rayon: Data Parallelism

```rust
use rayon::prelude::*;

let data: Vec<u64> = (0..1_000_000).collect();

// Sequential
let sum: u64 = data.iter().map(|&x| x * x).sum();

// Parallel — same API, just .par_iter()
let sum: u64 = data.par_iter().map(|&x| x * x).sum();

// Parallel sort
let mut v: Vec<i32> = (0..1000).rev().collect();
v.par_sort();

// Parallel map + filter + collect
let results: Vec<String> = names
    .par_iter()
    .filter(|n| n.starts_with('A'))
    .map(|n| format!("Hello, {n}!"))
    .collect();

// Spawn two parallel tasks
let (sum, product) = rayon::join(
    || data.par_iter().sum::<u64>(),
    || data.par_iter().product::<u64>(),
);
```

## Atomic Operations

For simple counters and flags without mutexes:

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use std::sync::Arc;

let counter = Arc::new(AtomicUsize::new(0));
let shutdown = Arc::new(AtomicBool::new(false));

// In threads:
counter.fetch_add(1, Ordering::Relaxed); // Increment
let count = counter.load(Ordering::Acquire); // Read
shutdown.store(true, Ordering::Release); // Signal

// Ordering cheatsheet:
// Relaxed: no synchronization, just atomicity (counters OK)
// Acquire: synchronize with Release store (pair with Release)
// Release: synchronize with Acquire load (pair with Acquire)
// SeqCst: total order across all threads (use when Acquire/Release aren't enough)
```

## Thread Pool

```rust
// Use rayon for data parallelism
// Use tokio for async I/O

// Custom pool with crossbeam:
struct ThreadPool {
    workers: Vec<thread::JoinHandle<()>>,
    tx: crossbeam::channel::Sender<Box<dyn FnOnce() + Send>>,
}

impl ThreadPool {
    fn new(n: usize) -> Self {
        let (tx, rx) = crossbeam::channel::bounded::<Box<dyn FnOnce() + Send>>(64);
        let rx = std::sync::Arc::new(rx);

        let workers = (0..n).map(|_| {
            let rx = Arc::clone(&rx);
            thread::spawn(move || {
                for task in rx.iter() {
                    task();
                }
            })
        }).collect();

        Self { workers, tx }
    }

    fn execute<F: FnOnce() + Send + 'static>(&self, f: F) {
        self.tx.send(Box::new(f)).unwrap();
    }
}
```

## Debugging Concurrency Issues

### Detecting Deadlocks

```rust
// Classic deadlock: A locks mutex1 then mutex2, B locks mutex2 then mutex1
// Fix: Always acquire locks in the same order
// Or: Use a single mutex for both, or restructure data

// Detect at runtime (parking_lot feature):
use parking_lot::deadlock;
thread::spawn(|| {
    loop {
        thread::sleep(Duration::from_secs(10));
        let deadlocks = deadlock::check_deadlock();
        if !deadlocks.is_empty() {
            eprintln!("{} deadlocks detected", deadlocks.len());
        }
    }
});
```

### Loom: Testing Concurrency

```rust
// Use loom for model-checking concurrent code
#[cfg(loom)]
use loom::sync::atomic::{AtomicUsize, Ordering};

#[cfg(loom)]
#[test]
fn test_concurrent_increment() {
    loom::model(|| {
        let counter = Arc::new(AtomicUsize::new(0));
        let c1 = Arc::clone(&counter);
        let c2 = Arc::clone(&counter);

        let t1 = loom::thread::spawn(move || c1.fetch_add(1, Ordering::SeqCst));
        let t2 = loom::thread::spawn(move || c2.fetch_add(1, Ordering::SeqCst));

        t1.join().unwrap();
        t2.join().unwrap();
        assert_eq!(counter.load(Ordering::SeqCst), 2);
    });
}
```

## Common Patterns

```rust
// Producer-consumer with backpressure
let (tx, rx) = crossbeam::channel::bounded(1000);

// Fan-out: one sender, multiple receivers
let (tx, rx) = crossbeam::channel::unbounded();
let receivers: Vec<_> = (0..4).map(|_| rx.clone()).collect();

// Fan-in: multiple senders, one receiver
let (tx, rx) = crossbeam::channel::unbounded();
let senders: Vec<_> = (0..4).map(|_| tx.clone()).collect();
drop(tx); // Close original

// One-shot: signal completion
let (done_tx, done_rx) = crossbeam::channel::bounded::<()>(1);
thread::spawn(move || {
    do_work();
    done_tx.send(()).unwrap();
});
done_rx.recv().unwrap(); // Wait for completion
```

## References

- [Rust Book: Fearless Concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html)
- [crossbeam](https://docs.rs/crossbeam)
- [rayon](https://docs.rs/rayon)
- [parking_lot](https://docs.rs/parking_lot) — faster mutex/rwlock
- [loom](https://docs.rs/loom) — concurrency model checker
