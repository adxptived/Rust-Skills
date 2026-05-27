# Lockless Concurrency: Atomic Primitives & Memory Barriers

Lockless programming avoids OS scheduling overhead by updating values directly via atomic CPU instructions.

## 1. Atomic Primitives

Atomic types (like `AtomicBool`, `AtomicUsize`, `AtomicI32`) live in `std::sync::atomic`.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

let active_connections = Arc::new(AtomicUsize::new(0));
let mut handles = vec![];

for _ in 0..5 {
    let conn = Arc::clone(&active_connections);
    let handle = thread::spawn(move || {
        conn.fetch_add(1, Ordering::Relaxed);
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

println!("Connections: {}", active_connections.load(Ordering::Relaxed));
```

## 2. Available Atomic Types

| Type | Width | Use case |
|------|-------|----------|
| `AtomicBool` | 1 byte | Flags, toggles, enable/disable |
| `AtomicI8` .. `AtomicI64` | 1-8 bytes | Signed counters, accumulators |
| `AtomicU8` .. `AtomicU64` | 1-8 bytes | Unsigned counters, indices |
| `AtomicUsize` | pointer width | Array indices, lengths, counts |
| `AtomicIsize` | pointer width | Signed pointer-width values |
| `AtomicPtr<T>` | pointer width | Lock-free linked lists, Treiber stacks |
| `AtomicI128` / `AtomicU128` | 16 bytes | 128-bit CAS (on supported platforms) |

## 3. Memory Orderings Explained

When reading or writing atomic variables, specify how CPU and compiler instruction reordering should behave:

| Ordering | Reordering guarantees | Performance | Use case |
|----------|----------------------|-------------|----------|
| `Relaxed` | No ordering constraints beyond atomicity | Fastest | Counters, statistics, flags with no dependencies |
| `Acquire` | Prevents subsequent reads/writes from moving before this load | Fast | Load in a load-acquire/store-release pair |
| `Release` | Prevents preceding reads/writes from moving after this store | Fast | Store in a load-acquire/store-release pair |
| `AcqRel` | Acquire for loads, release for stores (RMW only) | Moderate | read-modify-write operations |
| `SeqCst` | Strict total global ordering across all threads | Slowest | General safety, easiest to reason about |

### When to Use What

```rust
// Relaxed: fine for stats counters
let hits = HITS.fetch_add(1, Ordering::Relaxed);

// Acquire/Release pair: publish data safely
// Thread 1:
DATA.store(value, Ordering::Release);
READY.store(true, Ordering::Release);
// Thread 2:
if READY.load(Ordering::Acquire) {
    let val = DATA.load(Ordering::Acquire); // sees thread 1's write
}

// SeqCst: when you need total order across multiple atomics
// Use as default when unsure — optimize to weaker ordering only if profiled
FLAG.store(true, Ordering::SeqCst);
```

## 4. Compare-and-Swap (CAS) Loop

Implement thread-safe modifications by checking and updating states in loops.

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

pub fn update_max(val: &AtomicUsize, new_val: usize) {
    let mut current = val.load(Ordering::Relaxed);
    loop {
        if current >= new_val {
            break;
        }
        match val.compare_exchange_weak(current, new_val, Ordering::SeqCst, Ordering::Relaxed) {
            Ok(_) => break,
            Err(actual) => current = actual,
        }
    }
}
```

### compare_exchange vs compare_exchange_weak

| Method | Spurious failure | Performance | Use when |
|--------|-----------------|-------------|----------|
| `compare_exchange` | No | Slower | Need guaranteed behavior |
| `compare_exchange_weak` | Yes (on some platforms) | Faster (no LL/SC loop) | Retry loop is acceptable |

## 5. Fetch Operations Reference

```rust
use std::sync::atomic::Ordering;

// Add/subtract
counter.fetch_add(1, Ordering::Relaxed);     // x++
counter.fetch_sub(1, Ordering::Relaxed);     // x--

// Bitwise
flags.fetch_or(0b0010, Ordering::SeqCst);   // set bits
flags.fetch_and(!0b0001, Ordering::SeqCst); // clear bits
flags.fetch_xor(0b0100, Ordering::SeqCst);  // toggle bits

// Swap
let old = ptr.swap(new_ptr, Ordering::SeqCst); // store, return previous

// Boolean
let prev = flag.swap(true, Ordering::SeqCst);  // test-and-set

// Min/max (nightly only)
// val.fetch_min(n, Ordering::Relaxed);
// val.fetch_max(n, Ordering::Relaxed);
```

All fetch operations return the **previous** value before the operation.

## 6. Atomic Ordering Anti-Patterns

```rust
// Bad: insufficient ordering for dependent data
DATA.store(value, Ordering::Relaxed); // may not be visible to other threads

// Bad: acquire on a store (no-op — no preceding reads to constrain)
flag.store(true, Ordering::Acquire); // Acq on store = Relaxed in practice

// Bad: release on a load (no-op — no succeeding writes to constrain)
flag.load(Ordering::Release); // Rel on load = Relaxed in practice

// Good: match acquire/release pairs for data publication
DATA.store(value, Ordering::Release);
let v = DATA.load(Ordering::Acquire); // correct
```

## 7. Practical Lockless Data Structures

```rust
// Lock-free counter with peek
struct Metrics {
    requests: AtomicU64,
    errors: AtomicU64,
}

impl Metrics {
    fn record_request(&self, is_error: bool) {
        self.requests.fetch_add(1, Ordering::Relaxed);
        if is_error {
            self.errors.fetch_add(1, Ordering::Relaxed);
        }
    }

    fn snapshot(&self) -> (u64, u64) {
        // Order doesn't matter for metrics — relaxed is fine
        (self.requests.load(Ordering::Relaxed), self.errors.load(Ordering::Relaxed))
    }
}
```

```rust
// Lock-free once-init pattern
use std::sync::atomic::{AtomicBool, Ordering};
use std::cell::UnsafeCell;

struct LazyInit<T> {
    initialized: AtomicBool,
    data: UnsafeCell<Option<T>>,
}

impl<T> LazyInit<T> {
    fn get_or_init<F: FnOnce() -> T>(&self, f: F) -> &T {
        if !self.initialized.load(Ordering::Acquire) {
            self.init_slow(f);
        }
        unsafe { (*self.data.get()).as_ref().unwrap() }
    }

    fn init_slow<F: FnOnce() -> T>(&self, f: F) {
        unsafe { *self.data.get() = Some(f()) };
        self.initialized.store(true, Ordering::Release);
    }
}
// Note: for production use, use OnceLock<T> or once_cell
```

## 8. Hardware Considerations

- On x86, `Acquire`/`Release` ordering is essentially free (x86 is TSO).
- On ARM/PowerPC, weaker orderings provide real performance gains.
- `SeqCst` imposes full memory barrier — avoid in hot paths on ARM.
- Misaligned atomics may fault or silently misbehave — let the compiler align.
- 128-bit atomics (`AtomicI128`) use `cmpxchg16b` on x86 — slower than smaller atomics.
