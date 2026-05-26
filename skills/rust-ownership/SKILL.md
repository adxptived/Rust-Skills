---
name: rust-ownership
description: |
  Rust ownership, borrowing, and lifetimes — the core memory model. ALWAYS use this skill when:
  - User hits E0382 (use of moved value), E0505 (cannot move out of), E0506 (cannot assign to borrowed), E0507 (cannot move out of borrowed), E0515 (returning reference to local), E0597 (borrowed value does not live long enough)
  - User asks "why can't I use X after passing it to a function?"
  - User encounters "borrowed value does not live long enough"
  - User works with self-referential structs, Rc/Arc, or complex lifetime annotations
  - User asks about 'static, 'a, lifetime elision, or NLL (non-lexical lifetimes)
  - User clones excessively to appease the borrow checker
  - User asks about Copy vs Clone, move semantics, or references
  Trigger for ANY ownership/borrow/lifetime error or question.
---

# Rust Ownership & Lifetimes

> The borrow checker is a design tool, not an obstacle. When it pushes back, rethink data ownership.

## Core Mental Model

**Ownership = exclusive control over data lifetime.**
Every value has one owner. When the owner is dropped, the value is freed. No GC, no double-free.

**Ask before writing code:**
- Who *owns* this data?
- How long does it need to live?
- Who needs to *read* it vs *modify* it?

## Ownership Errors → Design Questions

| Error | Reflexive fix (wrong) | Right question |
|-------|----------------------|----------------|
| E0382 use of moved value | `.clone()` it | Who should own this? |
| E0597 doesn't live long enough | Add `'static` | Is the scope boundary wrong? |
| E0506 cannot assign to borrowed | Re-borrow mutably | Should mutation happen here? |
| E0507 move out of borrowed | `.clone()` the field | Should the caller give ownership? |
| E0515 return reference to local | Return owned type | Should caller pass in a buffer? |

## Borrowing Rules (Compile-time)

```
At any point in code: either
  - ONE mutable reference (&mut T), or
  - ANY NUMBER of immutable references (&T)
... but never both at the same time.
All references must be valid (no dangling pointers).
```

```rust
let mut data = vec![1, 2, 3];

// Multiple immutable borrows — OK
let r1 = &data;
let r2 = &data;
println!("{:?} {:?}", r1, r2); // r1 and r2 last used here

// Mutable borrow after immutable borrows END (NLL)
data.push(4); // OK: r1 and r2 no longer in use
```

## Copy vs Move vs Clone

```rust
// Copy types (stack-only, bitwise copy): i32, f64, bool, char, &T, [T; N] if T: Copy
let x: i32 = 5;
let y = x; // Copied, x still valid
println!("{x}"); // Fine

// Move types (heap data): String, Vec<T>, Box<T>
let s = String::from("hello");
let t = s; // MOVED — s is no longer valid
// println!("{s}"); // E0382!

// Clone: explicit deep copy
let s = String::from("hello");
let t = s.clone(); // Explicit copy, both valid
println!("{s} {t}");
```

**Rule of thumb**: If `Copy` doesn't make sense semantically (e.g., a file handle), use `Clone` only when you genuinely need two independent values.

## Borrowing vs Cloning

```rust
// Bad: Cloning just to pass to a read-only function
fn print_name(name: String) { println!("{name}"); }
print_name(user.name.clone()); // Wasteful

// Good: Borrow instead
fn print_name(name: &str) { println!("{name}"); }
print_name(&user.name); // Zero cost

// Good: Accept impl AsRef<str> for maximum flexibility
fn print_name(name: impl AsRef<str>) { println!("{}", name.as_ref()); }
print_name("literal");    // &str
print_name(&user.name);   // &String → auto-deref to &str
print_name(user.name);    // Owned String
```

## Lifetimes

Lifetimes tell the compiler how long references are valid. They're inferred in most cases (lifetime elision), but must be explicit when:
- Function returns a reference derived from multiple input references
- A struct holds a reference

```rust
// Elision works: compiler knows output comes from input
fn first(s: &str) -> &str {
    &s[..1] // lifetime of return = lifetime of s
}

// Must annotate: which input does the output borrow from?
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
// 'a = the shorter of x's and y's lifetime

// Struct holding a reference
struct Excerpt<'a> {
    text: &'a str, // Can't outlive the string it came from
}
```

### Common Lifetime Patterns

```rust
// 'static: lives for the entire program (string literals, leaked data)
fn get_greeting() -> &'static str {
    "Hello!" // String literals are 'static
}

// Returning owned value — avoids lifetime complexity
fn build_greeting(name: &str) -> String {
    format!("Hello, {name}!")
}

// Output lifetime from self (method pattern)
impl Cache {
    fn get(&self, key: &str) -> Option<&str> {
        // return lifetime = self's lifetime (elision rule 3)
        self.map.get(key).map(String::as_str)
    }
}
```

## Cow: Flexible Ownership

Use `Cow<'_, T>` when a function sometimes needs to allocate and sometimes doesn't:

```rust
use std::borrow::Cow;

fn normalize_username(name: &str) -> Cow<'_, str> {
    if name.chars().all(|c| c.is_lowercase()) {
        Cow::Borrowed(name)  // No allocation needed
    } else {
        Cow::Owned(name.to_lowercase())  // Allocates only when necessary
    }
}

// Caller doesn't care which variant they got
let n = normalize_username("Alice");
println!("{n}"); // Works for both Borrowed and Owned
```

## Interior Mutability

When you need mutation through a shared reference:

```rust
use std::cell::RefCell;
use std::sync::{Arc, Mutex};

// Single-threaded: RefCell (runtime borrow checking)
let data = RefCell::new(vec![1, 2, 3]);
data.borrow_mut().push(4); // Panics if already mutably borrowed

// Multi-threaded: Arc<Mutex<T>>
let data = Arc::new(Mutex::new(vec![1, 2, 3]));
let data2 = Arc::clone(&data);
std::thread::spawn(move || {
    data2.lock().unwrap().push(4);
});

// Read-heavy: Arc<RwLock<T>>
use std::sync::RwLock;
let data = Arc::new(RwLock::new(HashMap::new()));
let r = data.read().unwrap();   // Many readers at once
let mut w = data.write().unwrap(); // Exclusive writer
```

## Self-Referential Structs

Avoid self-referential structs — they're a borrow checker nightmare. Instead:

```rust
// Bad: struct referencing its own field
struct SelfRef {
    data: String,
    // ptr: &'??? str, // What lifetime? It references `data`!
}

// Good option 1: Store indices instead of references
struct Parser {
    input: String,
    pos: usize, // Position into input
}

// Good option 2: Pin + unsafe (for truly needed cases)
use std::pin::Pin;
// Use `pin-project` crate for safe pinned projections

// Good option 3: Separate lifetime (caller owns the data)
struct Parser<'a> {
    input: &'a str, // Caller's data, parser borrows it
    pos: usize,
}
```

## Quick Fixes for Common Errors

```rust
// E0382: Use of moved value
// Problem: moved into a closure, then used again
let name = String::from("Alice");
let greeting = move || println!("{name}");
// println!("{name}"); // ERROR

// Fix: clone before the move
let name = String::from("Alice");
let name2 = name.clone();
let greeting = move || println!("{name}");
println!("{name2}"); // OK

// E0502: Can't borrow mutably while immutably borrowed
let mut v = vec![1, 2, 3];
let first = v[0]; // Copy the value (i32 is Copy)
v.push(4);        // OK: no borrow active
println!("{first}");

// E0597: Borrowed value doesn't live long enough
// Problem: returning reference to local
fn bad() -> &str {
    let s = String::from("hello");
    &s // s dropped here, reference dangles!
}
// Fix: return owned
fn good() -> String {
    String::from("hello")
}
```

## Checklist: Before Cloning

- [ ] Does the callee actually need ownership, or just to read?
- [ ] Can I change the function signature to accept `&T` or `&str`?
- [ ] Could `Cow` eliminate the clone in the common path?
- [ ] Am I cloning in a loop? Consider restructuring ownership.
- [ ] Is this a design smell — does one value need to be in two places?

## References

- [The Rust Book — Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
- [The Rust Reference — Lifetimes](https://doc.rust-lang.org/reference/lifetimes.html)
- [Rustonomicon — Lifetimes in depth](https://doc.rust-lang.org/nomicon/lifetimes.html)
- [common-rust-lifetime-misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md) — essential reading
