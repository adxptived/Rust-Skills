# Concurrency Sync Primitives: Core Multi-Threading

Rust guarantees memory safety through type properties (`Send` and `Sync`) combined with synchronization primitive abstractions.

## 1. Thread Safety Invariants: Send & Sync

- **`Send`**: Indicates that ownership of the data type can be transferred across thread boundaries. Most types are `Send`. Examples of non-Send types include `Rc<T>` (since reference counters are not atomic).
- **`Sync`**: Indicates that it is safe to access the data type from multiple threads concurrently via shared references (`&T`). A type `T` is `Sync` if and only if `&T` is `Send`.

### Common Type Send/Sync Table

| Type | Send | Sync | Notes |
|------|------|------|-------|
| `i32`, `bool`, `Vec<T>` | Yes | Yes | Copy/move safe |
| `Rc<T>` | No | No | Non-atomic refcount |
| `Arc<T: Send + Sync>` | Yes | Yes | Atomic refcount |
| `Mutex<T>` | Yes | Yes | Lock + guard |
| `RwLock<T>` | Yes | Yes | Read/write lock |
| `RefCell<T>` | Yes | No | Runtime borrow check, no sync |
| `Cell<T>` | Yes | No | Interior mutability, no sync |
| `*const T`, `*mut T` | No | No | Raw pointers |
| `std::sync::OnceLock<T>` | Yes | Yes | One-time init |

## 2. Shared Ownership & Locks: Arc, Mutex, and RwLock

To share state across threads, wrap the synchronization lock in an `Arc`.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter_clone = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut guard = counter_clone.lock().unwrap();
        *guard += 1;
        // lock released automatically when guard falls out of scope
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

println!("Result: {}", *counter.lock().unwrap());
```

### RwLock Pattern

Use `RwLock` when you have many readers but few writers.

```rust
use std::sync::{Arc, RwLock};

let data = Arc::new(RwLock::new(vec![1, 2, 3]));

// Thread A (Reader)
{
    let read_guard = data.read().unwrap();
    println!("Read: {:?}", *read_guard);
} // read lock released

// Thread B (Writer)
{
    let mut write_guard = data.write().unwrap();
    write_guard.push(4);
} // write lock released
```

## 3. Mutex Guard Scoping

Hold mutex guards for the shortest duration necessary. The compiler drops them at the end of the scope.

```rust
// Bad: holding lock across an unrelated operation
let guard = cache.lock().unwrap();
let val = guard.get(&key).cloned();
let processed = expensive_process(val).await; // LOCK HELD HERE
drop(guard); // too late

// Good: scope the lock
let val = {
    let guard = cache.lock().unwrap();
    guard.get(&key).cloned()
}; // lock released here
let processed = expensive_process(val).await;
```

## 4. Poisoning

When a thread panics while holding a `Mutex`, the mutex becomes poisoned. Subsequent `.lock()` calls return `Err(PoisonError)`.

```rust
fn safe_lock(mtx: &Mutex<Data>) -> MutexGuard<'_, Data> {
    mtx.lock().unwrap_or_else(|poisoned| {
        eprintln!("detected poisoned mutex, recovering");
        poisoned.into_inner() // recover the value
    })
}

// Or use `.clear_poison()` on the mutex after recovery
```

## 5. Barrier for Synchronization

```rust
use std::sync::{Arc, Barrier};

let barrier = Arc::new(Barrier::new(3)); // 3 threads must reach barrier

let mut handles = vec![];
for i in 0..3 {
    let b = Arc::clone(&barrier);
    handles.push(thread::spawn(move || {
        // do work
        b.wait(); // block until all 3 threads arrive
        // proceed together
    }));
}
```

## 6. Condvar for Event Notification

```rust
use std::sync::{Arc, Condvar, Mutex};

let pair = Arc::new((Mutex::new(false), Condvar::new()));
let pair2 = Arc::clone(&pair);

thread::spawn(move || {
    let (lock, cvar) = &*pair2;
    let mut ready = lock.lock().unwrap();
    *ready = true;
    cvar.notify_one(); // wake up waiter
});

let (lock, cvar) = &*pair;
let mut ready = lock.lock().unwrap();
while !*ready {
    ready = cvar.wait(ready).unwrap(); // releases lock, reacquires on wake
}
```

## 7. OnceLock for Lazy Statics

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<AppConfig> = OnceLock::new();

fn get_config() -> &'static AppConfig {
    CONFIG.get_or_init(|| {
        AppConfig::load().expect("config must load")
    })
}
```

## 8. Lock-Free Pattern Guidance

| Primitive | Readers | Writers | Starvation risk | Best for |
|-----------|---------|---------|----------------|----------|
| `Mutex<T>` | 1 at a time | 1 at a time | None (fair) | Short critical sections |
| `RwLock<T>` | Many | 1 at a time | Writer starvation with many readers | Read-heavy workloads |
| `Atomic*` | Many | Many (CAS) | None for CAS | Counters, flags, metrics |
| `Barrier` | N/A | N/A | N/A | Sync point for N threads |
| `Condvar` | N/A | N/A | Missed wakeup | Blocking notification |
| `OnceLock<T>` | Many after init | 1 (init only) | N/A | Lazy singletons |

## 9. Deadlock Avoidance

- Never acquire locks in different orders across threads.
- Prefer `try_lock` with backoff in scenarios where lock order is hard to guarantee.
- Use `parking_lot` crate for deadlock detection (`PARKING_LOT_DEADLOCK_DETECTION=1`).

```rust
// Deadlock-prone: lock A then B in one thread, B then A in another
// Fix: always lock in the same order
fn transfer(a: &Mutex<Account>, b: &Mutex<Account>, amount: u64) {
    let ptr_a = &a as *const _ as usize;
    let ptr_b = &b as *const _ as usize;
    let (first, second) = if ptr_a < ptr_b { (a, b) } else { (b, a) };
    let mut a_guard = first.lock().unwrap();
    let mut b_guard = second.lock().unwrap();
    // transfer
}
```
