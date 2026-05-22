# Ownership, Borrowing, and Lifetimes

Deep dive into Rust's core memory management concepts.

## Table of Contents
1. [Ownership Rules](#ownership-rules)
2. [Move Semantics](#move-semantics)
3. [Borrowing](#borrowing)
4. [Lifetimes](#lifetimes)
5. [Common Patterns](#common-patterns)
6. [Troubleshooting](#troubleshooting)

## Ownership Rules

Three fundamental rules:

1. **Each value has exactly one owner** - A variable that holds the value
2. **When owner goes out of scope, value is dropped** - Memory freed, destructors run
3. **Ownership can be transferred (moved)** - New owner, old one invalid

```rust
{
    let s = String::from("hello"); // s owns the String
    // s is valid here
} // s goes out of scope, String dropped, memory freed
```

## Move Semantics

Types that don't implement `Copy` are moved, not copied:

```rust
// Move: s1 transfers ownership to s2
let s1 = String::from("hello");
let s2 = s1;
// s1 is now invalid - cannot be used

// Clone: explicit deep copy
let s1 = String::from("hello");
let s2 = s1.clone();
// Both s1 and s2 are valid

// Copy types (stack-only, bitwise copy)
let x = 5;
let y = x;
// Both x and y are valid (i32 implements Copy)
```

### Copy vs Clone

| Copy | Clone |
|------|-------|
| Implicit, automatic | Explicit, must call `.clone()` |
| Stack-only, bitwise | Can allocate, deep copy |
| `i32`, `f64`, `bool`, `char` | `String`, `Vec<T>`, `HashMap` |
| Can't customize behavior | Can customize via trait impl |

Types that implement `Copy`:
- All integer types (`i8`, `u32`, `isize`, etc.)
- Floating point (`f32`, `f64`)
- Booleans (`bool`)
- Characters (`char`)
- Tuples of `Copy` types: `(i32, i32)` is Copy, `(i32, String)` is not
- Arrays of `Copy` types with known size

## Borrowing

References allow using values without taking ownership.

### Immutable References (`&T`)

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
} // s goes out of scope, but doesn't own the String, nothing dropped

let s1 = String::from("hello");
let len = calculate_length(&s1); // Borrow s1
// s1 is still valid
```

**Multiple immutable references allowed:**
```rust
let s = String::from("hello");
let r1 = &s;
let r2 = &s;
let r3 = &s;
println!("{} {} {}", r1, r2, r3); // OK
```

### Mutable References (`&mut T`)

```rust
fn change(s: &mut String) {
    s.push_str(", world");
}

let mut s = String::from("hello");
change(&mut s);
```

**Only ONE mutable reference at a time:**
```rust
let mut s = String::from("hello");

let r1 = &mut s;
let r2 = &mut s; // ERROR: second mutable borrow

println!("{}, {}", r1, r2);
```

**Cannot mix mutable and immutable:**
```rust
let mut s = String::from("hello");

let r1 = &s;     // immutable borrow
let r2 = &s;     // immutable borrow
let r3 = &mut s; // ERROR: mutable borrow while immutable exists

println!("{}, {}, {}", r1, r2, r3);
```

### Non-Lexical Lifetimes (NLL)

References' lifetimes end at last use, not scope end:

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &s;
println!("{} {}", r1, r2);
// r1 and r2 no longer used after this

let r3 = &mut s; // OK: immutable borrows ended
println!("{}", r3);
```

### Reborrowing

Mutable references can be temporarily reborrowed:

```rust
fn foo(x: &mut i32) {
    *x += 1;
}

fn bar(x: &mut i32) {
    foo(x);      // Reborrow: &mut *x
    foo(x);      // Can reborrow again
    *x += 10;    // Original reference still valid
}
```

## Lifetimes

Lifetimes ensure references are valid for their entire usage.

### Lifetime Annotations

```rust
// Explicit lifetime: return value lives as long as both inputs
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

### Lifetime Elision Rules

Compiler infers lifetimes when unambiguous:

1. **Each reference parameter gets its own lifetime**
```rust
fn foo(x: &str, y: &str) 
// becomes
fn foo<'a, 'b>(x: &'a str, y: &'b str)
```

2. **If one input lifetime, output gets same lifetime**
```rust
fn foo(x: &str) -> &str
// becomes
fn foo<'a>(x: &'a str) -> &'a str
```

3. **If method with &self, output gets self's lifetime**
```rust
impl Foo {
    fn method(&self, x: &str) -> &str
    // becomes
    fn method<'a, 'b>(&'a self, x: &'b str) -> &'a str
}
```

### Struct Lifetimes

Structs containing references need lifetime annotations:

```rust
struct Excerpt<'a> {
    part: &'a str,
}

impl<'a> Excerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
    
    fn announce(&self, announcement: &str) -> &str {
        println!("Attention: {}", announcement);
        self.part // Returns reference with 'a lifetime
    }
}
```

### Static Lifetime

`'static` means the reference lives for entire program:

```rust
let s: &'static str = "I live forever"; // String literal

// Static in trait bounds: type owns data or has 'static refs
fn print_it(input: impl std::fmt::Display + 'static) {
    println!("{}", input);
}
```

### Lifetime Bounds

```rust
// T must outlive 'a
fn foo<'a, T: 'a>(x: &'a T) { ... }

// T must be 'static (owns its data)
fn bar<T: 'static>(x: T) { ... }

// Multiple lifetimes with bounds
fn complex<'a, 'b: 'a>(x: &'a str, y: &'b str) -> &'a str {
    // 'b outlives 'a, so y can be returned where x is expected
    if x.len() > 0 { x } else { y }
}
```

## Common Patterns

### Returning References

```rust
// Return reference to input data
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[..i];
        }
    }
    &s[..]
}

// Cannot return reference to local data
fn dangle() -> &String { // ERROR
    let s = String::from("hello");
    &s // s dropped here, reference invalid
}

// Return owned data instead
fn no_dangle() -> String {
    let s = String::from("hello");
    s // Ownership transferred out
}
```

### Interior Mutability

When you need mutation through immutable reference:

```rust
use std::cell::{Cell, RefCell};

struct Counter {
    // Cell for Copy types
    count: Cell<u32>,
    // RefCell for non-Copy types
    log: RefCell<Vec<String>>,
}

impl Counter {
    fn increment(&self) { // Note: &self, not &mut self
        self.count.set(self.count.get() + 1);
        self.log.borrow_mut().push("incremented".to_string());
    }
}
```

### Splitting Borrows

Borrow different struct fields independently:

```rust
struct Data {
    a: Vec<i32>,
    b: Vec<i32>,
}

impl Data {
    fn process(&mut self) {
        // Can borrow a and b mutably simultaneously
        let a = &mut self.a;
        let b = &mut self.b;
        
        a.push(b.pop().unwrap_or(0));
    }
}
```

## Troubleshooting

### Error: "borrowed value does not live long enough"

```rust
// Problem
fn example() -> &str {
    let s = String::from("hello");
    &s // ERROR: s dropped at end of function
}

// Solutions
fn example() -> String {
    String::from("hello") // Return owned
}

fn example() -> &'static str {
    "hello" // Return static string literal
}
```

### Error: "cannot borrow as mutable more than once"

```rust
// Problem
let mut v = vec![1, 2, 3];
let first = &mut v[0];
let second = &mut v[1]; // ERROR

// Solution 1: Use indices
let first_idx = 0;
let second_idx = 1;
v[first_idx] += 1;
v[second_idx] += 1;

// Solution 2: split_at_mut
let mut v = vec![1, 2, 3];
let (left, right) = v.split_at_mut(1);
let first = &mut left[0];
let second = &mut right[0];
```

### Error: "cannot move out of borrowed content"

```rust
// Problem
fn example(opt: &Option<String>) {
    match opt {
        Some(s) => println!("{}", s), // Can't move out of ref
        None => {}
    }
}

// Solution 1: Match on reference
fn example(opt: &Option<String>) {
    match opt {
        Some(ref s) => println!("{}", s), // Borrow instead
        None => {}
    }
}

// Solution 2: Use as_ref()
fn example(opt: &Option<String>) {
    if let Some(s) = opt.as_ref() {
        println!("{}", s);
    }
}
```
