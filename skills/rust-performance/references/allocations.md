# Reducing Allocations in Rust

Strategies for minimizing heap allocations and improving memory efficiency.

## Understanding Allocations

### Stack vs Heap

```rust
// Stack allocated - fast, automatic cleanup
let x: i32 = 42;
let array: [u8; 1024] = [0; 1024];

// Heap allocated - flexible size, manual tracking
let vec: Vec<u8> = vec![0; 1024];
let string: String = String::from("hello");
let boxed: Box<[u8; 1024]> = Box::new([0; 1024]);
```

### Common Allocation Sources

```rust
// Each creates heap allocation:
String::from("hello")           // String buffer
"hello".to_string()             // String buffer
vec![1, 2, 3]                   // Vec buffer
Box::new(value)                 // Boxed value
Arc::new(value)                 // Arc + value
HashMap::new()                  // Hash table
format!("x = {}", x)            // String buffer
collect::<Vec<_>>()             // Vec buffer
clone()                         // Duplicates data
```

## String Optimizations

### &str vs String

```rust
// Allocates - avoid if possible
fn greet(name: String) {
    println!("Hello, {}!", name);
}

// Zero allocation
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

// Flexible - accepts both
fn greet(name: impl AsRef<str>) {
    println!("Hello, {}!", name.as_ref());
}
```

### Cow<str> for Maybe-Owned

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.chars().all(|c| c.is_lowercase()) {
        // No allocation - return borrowed
        Cow::Borrowed(input)
    } else {
        // Allocation needed
        Cow::Owned(input.to_lowercase())
    }
}

// Usage
let a = normalize("hello");     // Borrowed, no alloc
let b = normalize("HELLO");     // Owned, allocates
```

### String Building

```rust
// Bad: Many allocations
let mut s = String::new();
for i in 0..100 {
    s = s + &i.to_string();  // New String each iteration!
}

// Better: Pre-allocate and reuse
let mut s = String::with_capacity(300);
for i in 0..100 {
    use std::fmt::Write;
    write!(s, "{}", i).unwrap();  // Writes to existing buffer
}

// Best for joining: collect
let s: String = (0..100).map(|i| i.to_string()).collect();

// Or use join
let parts: Vec<String> = (0..100).map(|i| i.to_string()).collect();
let s = parts.join(", ");
```

### SmallString / CompactString

```rust
use compact_str::CompactString;  // cargo add compact_str

// Stores small strings inline (up to 24 bytes on 64-bit)
let small: CompactString = CompactString::from("hello");  // No heap alloc
let large: CompactString = CompactString::from("this is a longer string"); // Heap
```

## Vec Optimizations

### Pre-allocation

```rust
// Bad: Multiple reallocations as it grows
let mut v = Vec::new();
for i in 0..1000 {
    v.push(i);  // May reallocate multiple times
}

// Good: Single allocation
let mut v = Vec::with_capacity(1000);
for i in 0..1000 {
    v.push(i);
}

// Best: From iterator
let v: Vec<_> = (0..1000).collect();
```

### Reusing Buffers

```rust
struct Processor {
    buffer: Vec<u8>,
}

impl Processor {
    fn process(&mut self, data: &[u8]) -> &[u8] {
        self.buffer.clear();  // Keeps capacity
        self.buffer.extend_from_slice(data);
        self.buffer.extend_from_slice(b"_processed");
        &self.buffer
    }
}

// Buffer reused across calls
let mut p = Processor { buffer: Vec::with_capacity(1024) };
p.process(b"data1");  // No allocation (fits in capacity)
p.process(b"data2");  // No allocation
```

### SmallVec

```rust
use smallvec::{SmallVec, smallvec};

// Stores up to N elements inline, spills to heap after
type SmallItems = SmallVec<[Item; 4]>;

let mut items: SmallItems = smallvec![];
items.push(item1);  // Inline, no heap
items.push(item2);  // Inline
items.push(item3);  // Inline
items.push(item4);  // Inline
items.push(item5);  // Spills to heap
```

### ArrayVec (Fixed Capacity)

```rust
use arrayvec::ArrayVec;

// Stack-only, panics if exceeded
let mut arr: ArrayVec<i32, 4> = ArrayVec::new();
arr.push(1);
arr.push(2);
// arr.try_push(5)?; // Returns Err if full
```

## HashMap Optimizations

### Pre-sizing

```rust
use std::collections::HashMap;

// Bad: Rehashes as it grows
let mut map = HashMap::new();

// Good: Pre-allocate
let mut map = HashMap::with_capacity(expected_size);
```

### Entry API

```rust
// Bad: Double lookup
if !map.contains_key(&key) {
    map.insert(key, compute_value());
}

// Good: Single lookup
map.entry(key).or_insert_with(|| compute_value());

// Modify existing or insert
*map.entry(key).or_insert(0) += 1;
```

### Faster Hashers

```rust
use rustc_hash::FxHashMap;  // cargo add rustc-hash

// Faster for integer keys
let mut map: FxHashMap<u64, Value> = FxHashMap::default();

// Or use ahash
use ahash::AHashMap;
let mut map: AHashMap<String, Value> = AHashMap::new();
```

## Avoiding Clone

### Borrow Instead

```rust
// Bad: Clones unnecessarily
fn process(items: &[Item]) {
    for item in items {
        let owned = item.clone();  // Why clone?
        do_something(&owned);
    }
}

// Good: Borrow
fn process(items: &[Item]) {
    for item in items {
        do_something(item);
    }
}
```

### Take Ownership When Needed

```rust
// If you need ownership, take it
fn process(items: Vec<Item>) -> Vec<Output> {
    items.into_iter()  // Consumes items
        .map(transform)
        .collect()
}

// Instead of
fn process(items: &[Item]) -> Vec<Output> {
    items.iter()
        .map(|i| i.clone())  // Unnecessary clone
        .map(transform)
        .collect()
}
```

### Rc/Arc for Shared Ownership

```rust
use std::sync::Arc;

#[derive(Clone)]
struct ExpensiveData {
    data: Arc<Vec<u8>>,  // Cheap to clone
}

impl ExpensiveData {
    fn new(data: Vec<u8>) -> Self {
        Self { data: Arc::new(data) }
    }
}

// Cloning just increments reference count
let a = ExpensiveData::new(vec![0; 1_000_000]);
let b = a.clone();  // Cheap! Just Arc clone
```

## Iterator Optimizations

### Avoid Intermediate Collections

```rust
// Bad: Creates intermediate Vec
let processed: Vec<_> = items.iter()
    .filter(|x| x.is_valid())
    .collect();
let sum: i32 = processed.iter().sum();

// Good: Lazy evaluation
let sum: i32 = items.iter()
    .filter(|x| x.is_valid())
    .sum();
```

### Use chunks/windows

```rust
// Process in batches without allocation
for chunk in data.chunks(1024) {
    process_batch(chunk);
}

// Sliding window
for window in data.windows(3) {
    let [a, b, c] = window else { continue };
    // process triple
}
```

## Boxing Strategies

### Box Only When Needed

```rust
// Usually unnecessary
let boxed: Box<Config> = Box::new(Config::default());

// Necessary: recursive types
enum Tree {
    Leaf(i32),
    Node(Box<Tree>, Box<Tree>),
}

// Necessary: trait objects
let handlers: Vec<Box<dyn Handler>> = vec![];

// Necessary: large types on stack
fn process() {
    let big: Box<[u8; 1_000_000]> = Box::new([0; 1_000_000]);
}
```

### Inline Small Types

```rust
// Small type - inline in enum variant
enum Value {
    Int(i64),
    Float(f64),
    Bool(bool),
    String(Box<str>),  // Only box the large one
}
```

## Memory Pools

### Typed Arena

```rust
use typed_arena::Arena;

let arena: Arena<Node> = Arena::new();

// All allocations from same pool
let node1 = arena.alloc(Node::new(1));
let node2 = arena.alloc(Node::new(2));
// Nodes live as long as arena

// All freed at once when arena dropped
```

### Object Pool

```rust
use std::sync::Mutex;

struct Pool<T> {
    items: Mutex<Vec<T>>,
    factory: fn() -> T,
}

impl<T> Pool<T> {
    fn get(&self) -> PoolGuard<T> {
        let item = self.items.lock().unwrap()
            .pop()
            .unwrap_or_else(|| (self.factory)());
        PoolGuard { pool: self, item: Some(item) }
    }
}

struct PoolGuard<'a, T> {
    pool: &'a Pool<T>,
    item: Option<T>,
}

impl<T> Drop for PoolGuard<'_, T> {
    fn drop(&mut self) {
        if let Some(item) = self.item.take() {
            self.pool.items.lock().unwrap().push(item);
        }
    }
}
```

## Quick Reference

| Pattern | Use Case |
|---------|----------|
| `&str` over `String` | Function parameters |
| `Cow<str>` | Maybe-modify strings |
| `with_capacity` | Known collection size |
| `SmallVec<[T; N]>` | Usually-small arrays |
| `ArrayVec<T, N>` | Fixed max size, stack only |
| `FxHashMap` | Integer keys |
| `Arc<T>` | Shared immutable data |
| Reuse buffers | Hot loops |
| Arena | Many short-lived allocations |
