---
name: rust-memory-optimization
description: |
  Memory optimization and allocation reduction in Rust. Use when:
  - User is designing memory-sensitive or low-latency systems
  - Optimizing data structures to reduce memory footprint
  - Replacing heap allocations with stack-allocated/bounded structures (SmallVec, ArrayVec, ThinVec)
  - Handling small strings with optimizations (CompactString)
  - Reusing collections via clear() in loops
  - Implementing zero-copy decoding and references
  - Structuring arena-allocated memory (Bumpalo)
  - Measuring memory usage, profiling heap, or diagnosing leaks
---



## Quick Navigation

- [references/stack_allocations.md](references/stack_allocations.md)
- [references/arenas.md](references/arenas.md)

# Rust Memory Optimization Guide

Advanced strategies for reducing allocations, shrinking layout sizes, and boosting performance.

## Core Rules & Patterns

### 1. Bounded Heap-Free Collections
For collections that are usually small, avoid standard vectors to bypass heap allocations.
- **`SmallVec`**: Stack-allocated up to a threshold, overflows to the heap.
- **`ArrayVec`**: Fixed-capacity array wrapper; never allocates on the heap.
- **`ThinVec`**: Only takes a single pointer size when empty, reducing size in enum layouts.

```rust
use smallvec::{smallvec, SmallVec};
use arrayvec::ArrayVec;

// SmallVec: Allocates 4 items on the stack. Goes to heap only if count > 4.
let mut local_tags: SmallVec<[String; 4]> = smallvec![];
local_tags.push("Rust".to_string());

// ArrayVec: Absolute limit of 8 items on the stack. No heap overflow.
let mut coordinates: ArrayVec<f64, 8> = ArrayVec::new();
coordinates.push(42.0);
```

### 2. Small String Optimization (SSO)
Standard `String` consumes 24 bytes and allocates heap space immediately. Use `CompactString` for inline small strings (under 24 bytes) without allocating.

```rust
use compact_str::CompactString;

// No heap allocation for strings <= 24 characters
let username = CompactString::new("user_123");
```

### 3. Boxed Slices vs Vectors
If a collection's size is determined once at initialization and never changes, convert it from `Vec<T>` to `Box<[T]>` to save 8 bytes of stack capacity space.

```rust
let items_vec: Vec<i32> = vec![1, 2, 3];

// Save capacity field (shrinks stack representation from 24 to 16 bytes)
let items_slice: Box<[i32]> = items_vec.into_boxed_slice();
```

### 4. Zero-Copy Patterns
Instead of cloning or allocating strings, hold references to existing memory using slices or crates like `bytes::Bytes`.

```rust
use bytes::Bytes;

pub struct Message {
    // Zero-copy reference-counted byte buffer slice
    pub payload: Bytes,
}

// Can be sliced and shared among threads without cloning underlying memory
let original = Bytes::from(vec![0; 1024]);
let slice = original.slice(0..100); 
```

### 5. Arena Allocation
For batch allocation where structures share a lifecycle (e.g. AST parsing), use arena allocators (`bumpalo`) to allocate items sequentially in a continuous region of memory and free them all at once.

```rust
use bumpalo::Bump;

let bump = Bump::new();

// Allocates quickly on the bump arena
let string_ref = bump.alloc_str("dynamic string");
let int_ref = bump.alloc(42);

// Entire arena is cleared at once on drop
```

### 6. Collection Reuse
Reuse collection allocations in performance-critical loops with `.clear()` instead of re-instantiating.

```rust
// Bad: Allocates a new vector every iteration
for item in feed {
    let mut batch = Vec::new();
    batch.push(item);
}

// Good: Reuses allocation buffer
let mut batch = Vec::with_capacity(100);
for item in feed {
    batch.clear();
    batch.push(item);
}
```

## Layout Awareness

Use `std::mem::size_of` to understand data layout.

```rust
println!("User size: {}", std::mem::size_of::<User>());
```

Reorder fields to reduce padding only when the type is numerous or hot.

```rust
// Often smaller: group large-alignment fields first
struct Record {
    id: u64,
    flags: u32,
    kind: u8,
}
```

## String Strategy

```rust
fn label<'a>(name: &'a str, fallback: &'a str) -> std::borrow::Cow<'a, str> {
    if name.is_empty() { fallback.into() } else { name.into() }
}
```

Avoid `format!` in tight loops. Reuse buffers with `clear()`.

## Zero-Copy Parsing

Return slices into the input when the input outlives parsed data.

```rust
pub struct Header<'a> {
    pub name: &'a str,
    pub value: &'a str,
}
```

Do not force zero-copy when it creates unmanageable lifetimes for little gain.

## ThinVec for Enum Payloads

When enums contain `Vec` variants, `ThinVec` reduces the enum size by storing only a pointer when empty:

```rust
use thin_vec::ThinVec;

// Regular Vec in an enum makes the enum 24+ bytes even when empty
enum Node {
    Leaf(i64),
    Children(Vec<Node>),     // 24 bytes for Vec
}

// ThinVec only stores 8 bytes (a pointer) when empty
enum NodeOptimized {
    Leaf(i64),
    Children(ThinVec<Node>), // 8 bytes for ThinVec
}
```

## Cow for Conditional Ownership

```rust
use std::borrow::Cow;

pub struct Error {
    message: Cow<'static, str>,
}

impl Error {
    // No allocation for static messages
    pub fn new(msg: &'static str) -> Self {
        Self { message: Cow::Borrowed(msg) }
    }

    // Allocates only when dynamic
    pub fn with_detail(msg: String) -> Self {
        Self { message: Cow::Owned(msg) }
    }
}
```

## Arc vs Rc Trade-offs

```rust
use std::rc::Rc;
use std::sync::Arc;

// Single-threaded shared ownership — cheaper than Arc
let shared = Rc::new(large_data);

// Multi-threaded shared ownership — required across threads
let shared = Arc::new(large_data);
```

Prefer `Rc` when data never crosses thread boundaries. `Arc` adds atomic reference counting overhead.

## HashMap Memory Optimization

```rust
use hashbrown::HashMap;

// hashbrown's HashMap uses less memory than std's and is faster
let mut map: HashMap<String, i32> = HashMap::with_capacity(128);

// Reserve capacity upfront to avoid rehashing
map.reserve(expected_count);

// Consider FxHashMap for non-adversarial keys
use rustc_hash::FxHashMap;
let mut fast_map: FxHashMap<String, i32> = FxHashMap::default();
```

## Stack vs Heap Decision Matrix

| Type | Stack Size | Heap Alloc | Use When |
|------|-----------|------------|----------|
| `String` | 24 bytes | Yes | Owns mutable text |
| `&str` | 16 bytes | No | Borrowed text, lifetime bound |
| `CompactString` | 24 bytes | No (≤24 chars) | Short strings that own data |
| `Cow<str>` | 32 bytes | Conditionally | Mostly static, rarely dynamic |
| `Box<str>` | 16 bytes | Yes | Owned fixed string |
| `Vec<T>` | 24 bytes | Yes | Growable collection |
| `SmallVec<[T; N]>` | ~24+ bytes | Only if >N | Usually small, occasionally large |
| `Box<[T]>` | 16 bytes | Yes | Fixed-size after construction |
| `ThinVec<T>` | 8 bytes | Only if non-empty | Empty or small in enums |

## Measuring Memory

```rust
use std::alloc::{GlobalAlloc, System, Layout};

struct TrackingAllocator;

unsafe impl GlobalAlloc for TrackingAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        System.alloc(layout)
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        System.dealloc(ptr, layout)
    }
}

// For detailed profiling, use dhat, heaptrack, or valgrind
```

Use `dhat` crate for heap profiling in tests:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn memory_test() {
        let _profiler = dhat::Profiler::new_heap();
        // code under test
        // dhat prints allocation stats on drop
    }
}
```

## Optimization Workflow

1. Measure allocations and CPU first.
2. Identify the hot path and expected data sizes.
3. Pick the narrowest memory change.
4. Benchmark before/after.
5. Keep readability unless the win is meaningful.

## Allocation Decision Rules

- `Vec<T>`: general purpose growable collection.
- `SmallVec<[T; N]>`: usually-small collection with occasional heap spill.
- `ArrayVec<T, N>`: hard maximum size, no heap allocation.
- `Box<[T]>`: fixed-size heap slice after construction.
- `Bytes`: shared immutable byte buffers.
- `Bump`: many same-lifetime allocations.
- `Cow<'a, T>`: borrow unless mutation/ownership is required.

## Anti-Patterns

```rust
// Bad: cloning a large Vec just to avoid lifetime complexity
let data = original.clone(); // expensive and unnecessary

// Good: use Arc or Cow for shared ownership
let data: Arc<[u8]> = original.into();
```

```rust
// Bad: allocating inside a hot loop
for item in items {
    let s = format!("processing {}", item); // allocates every iteration
}

// Good: reuse a buffer
let mut buf = String::with_capacity(128);
for item in items {
    buf.clear();
    write!(buf, "processing {}", item).unwrap();
}
```

- Replacing every `Vec` with `SmallVec` without measuring.
- Keeping arena references after reset.
- Cloning strings to satisfy ownership instead of shortening borrow scopes.
- Using `Arc` as a default clone workaround.
- Optimizing layout for types created a handful of times.
- Using `Box<dyn Trait>` when generics would eliminate the allocation entirely.

## Review Prompt

When reviewing memory optimization, ask what was measured, which allocation is removed, what lifetime assumptions changed, and whether the new type increases stack size or API complexity.

## Optimization Checklist

- Measure allocations before changing data structures.
- Preallocate collections only when capacity is known or bounded.
- Prefer borrowing and `Cow` before cloning strings or buffers.
- Use arenas only when lifetimes are truly batch-scoped.
- Check stack growth when replacing heap allocations with inline storage.
- Re-benchmark after changes and keep the simpler version if gains are noise.
- Profile with `dhat` or `heaptrack` to find actual allocation hotspots.
- Consider `hashbrown` or `rustc-hash` for HashMap-heavy code.

## References

- [Rust Performance Book: Heap Allocations](https://nnethercote.github.io/perf-book/heap-allocations.html)
- [smallvec](https://docs.rs/smallvec)
- [arrayvec](https://docs.rs/arrayvec)
- [bumpalo](https://docs.rs/bumpalo)
- [compact_str](https://docs.rs/compact_str)
- [bytes](https://docs.rs/bytes)
- [dhat](https://docs.rs/dhat)
- [hashbrown](https://docs.rs/hashbrown)
