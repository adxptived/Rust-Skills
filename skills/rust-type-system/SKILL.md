---
name: rust-type-system
description: |
  Rust type-driven design — make invalid states unrepresentable. Use this skill when:
  - User asks about newtype pattern, wrapper types, or branded types
  - User implements state machines, typestate pattern, or compile-time validation
  - User uses PhantomData, marker types, ZST, or phantom types
  - User asks "how do I prevent misuse of my API at compile time?"
  - User works with sealed traits, non-exhaustive enums, or repr(transparent)
  - User wants to encode constraints (validated email, positive integers, non-empty Vec)
  - User asks about generic type parameters, associated types, or GATs
  - User implements the builder pattern with mandatory fields
  Trigger when the user wants the compiler to enforce invariants.
---

# Rust Type-Driven Design

> Make invalid states unrepresentable. If the compiler can catch it, don't leave it to runtime.

## Core Question

Before reaching for `if` checks or runtime panics:
- Can the compiler reject invalid inputs?
- Can invalid states be unrepresentable in the type system?
- Can I validate at construction and trust invariants everywhere else?

## Newtype Pattern

Wrap primitives to create distinct, incompatible types:

```rust
// Without newtypes: easy to swap arguments
fn transfer(from: u64, to: u64, amount: u64) { ... }
transfer(order_id, user_id, cents); // Compiles! Bug at runtime.

// With newtypes: compiler catches swaps
struct UserId(u64);
struct OrderId(u64);
struct Cents(u64);

fn transfer(from: UserId, to: UserId, amount: Cents) { ... }
transfer(order_id, user_id, amount); // E0308: mismatched types ✓
```

### Newtype with Validation (Parse, Don't Validate)

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Email(String);

impl Email {
    pub fn new(s: &str) -> Result<Self, EmailError> {
        // Validate ONCE at the boundary
        if s.contains('@') && s.split('@').nth(1).map_or(false, |d| d.contains('.')) {
            Ok(Email(s.to_lowercase()))
        } else {
            Err(EmailError::Invalid(s.to_string()))
        }
    }

    pub fn as_str(&self) -> &str { &self.0 }
}

// Deref for ergonomic access to String methods
impl std::ops::Deref for Email {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}

// Now: every Email is KNOWN to be valid. No re-checking needed.
fn send_email(to: &Email, body: &str) { /* trust it's valid */ }
```

### Non-Empty Collections

```rust
pub struct NonEmpty<T> {
    head: T,
    tail: Vec<T>,
}

impl<T> NonEmpty<T> {
    pub fn new(head: T) -> Self { Self { head, tail: vec![] } }

    pub fn push(&mut self, item: T) { self.tail.push(item); }

    // Infallible — always returns something (unlike Vec::first())
    pub fn first(&self) -> &T { &self.head }

    pub fn len(&self) -> usize { 1 + self.tail.len() }
}
```

## Typestate Pattern

Encode state transitions in the type system. Invalid transitions don't compile.

```rust
// State marker types (zero-sized, no runtime cost)
struct Draft;
struct PendingReview;
struct Published;

struct Post<State> {
    content: String,
    _state: std::marker::PhantomData<State>,
}

// Only Draft posts can be edited
impl Post<Draft> {
    pub fn new(content: String) -> Self {
        Self { content, _state: std::marker::PhantomData }
    }

    pub fn edit(&mut self, content: String) {
        self.content = content;
    }

    pub fn submit(self) -> Post<PendingReview> {
        Post { content: self.content, _state: std::marker::PhantomData }
    }
}

// Only PendingReview posts can be published or rejected
impl Post<PendingReview> {
    pub fn approve(self) -> Post<Published> {
        Post { content: self.content, _state: std::marker::PhantomData }
    }
    pub fn reject(self) -> Post<Draft> {
        Post { content: self.content, _state: std::marker::PhantomData }
    }
}

// Only Published posts can be read
impl Post<Published> {
    pub fn content(&self) -> &str { &self.content }
}

// Compile-time enforcement:
let post = Post::new("Hello".into());
post.edit("Updated".into()); // OK
let post = post.submit();
// post.edit("No!"); // ERROR: Post<PendingReview> has no edit()
let post = post.approve();
println!("{}", post.content()); // OK
```

## Typestate Builder (Required Fields at Compile Time)

```rust
// Marker types for builder state
struct NoUrl;
struct HasUrl(String);
struct NoMethod;
struct HasMethod(Method);

struct RequestBuilder<U, M> {
    url: U,
    method: M,
    headers: Vec<(String, String)>,
    timeout_secs: u64,
}

// Start with no required fields set
impl RequestBuilder<NoUrl, NoMethod> {
    pub fn new() -> Self {
        Self { url: NoUrl, method: NoMethod, headers: vec![], timeout_secs: 30 }
    }
}

// Set URL — transitions to HasUrl state
impl<M> RequestBuilder<NoUrl, M> {
    pub fn url(self, url: impl Into<String>) -> RequestBuilder<HasUrl, M> {
        RequestBuilder { url: HasUrl(url.into()), method: self.method,
                        headers: self.headers, timeout_secs: self.timeout_secs }
    }
}

// Set method — transitions to HasMethod state
impl<U> RequestBuilder<U, NoMethod> {
    pub fn method(self, method: Method) -> RequestBuilder<U, HasMethod> {
        RequestBuilder { url: self.url, method: HasMethod(method),
                        headers: self.headers, timeout_secs: self.timeout_secs }
    }
}

// Optional fields available in any state
impl<U, M> RequestBuilder<U, M> {
    pub fn header(mut self, key: impl Into<String>, val: impl Into<String>) -> Self {
        self.headers.push((key.into(), val.into()));
        self
    }
    pub fn timeout(mut self, secs: u64) -> Self {
        self.timeout_secs = secs;
        self
    }
}

// build() ONLY available when both URL and Method are set
impl RequestBuilder<HasUrl, HasMethod> {
    pub fn build(self) -> Request {
        Request { url: self.url.0, method: self.method.0,
                  headers: self.headers, timeout_secs: self.timeout_secs }
    }
}

// Usage:
let req = RequestBuilder::new()
    .url("https://api.example.com/data")
    .method(Method::GET)
    .header("Authorization", "Bearer token")
    .build(); // Compiles ✓

// RequestBuilder::new().build(); // E0599: no method build() ✓
```

## State Machine with Enums

For runtime state (when typestate is too rigid):

```rust
#[derive(Debug)]
pub enum Connection {
    Disconnected,
    Connecting { attempt: u32, started_at: std::time::Instant },
    Connected { session_id: String, established_at: std::time::Instant },
    Reconnecting { attempts: u32, last_error: String },
}

impl Connection {
    pub fn connect(&self) -> Result<Connection, ConnectError> {
        match self {
            Connection::Disconnected => Ok(Connection::Connecting {
                attempt: 1,
                started_at: std::time::Instant::now(),
            }),
            Connection::Connected { .. } => Err(ConnectError::AlreadyConnected),
            other => Err(ConnectError::InvalidState(format!("{other:?}"))),
        }
    }

    pub fn is_usable(&self) -> bool {
        matches!(self, Connection::Connected { .. })
    }
}

// Make illegal transitions explicit in code:
// You cannot call send() on a Disconnected connection
// because send() takes &Connection and does the state check.
```

## PhantomData

`PhantomData<T>` adds type information without runtime cost:

```rust
use std::marker::PhantomData;

// Token that is valid only for a specific user's session
struct Token<User> {
    value: String,
    _user: PhantomData<User>, // Zero bytes at runtime
}

struct AdminUser;
struct RegularUser;

fn admin_action(token: &Token<AdminUser>) { /* only admins */ }

// Token<RegularUser> cannot be passed to admin_action
// The types are distinct even though the struct is the same.
```

## Sealed Traits

Prevent external implementations — stable extension point without full openness:

```rust
mod private {
    pub trait Sealed {} // Private, so external crates can't implement it
}

pub trait DatabaseDriver: private::Sealed {
    fn execute(&self, query: &str) -> Result<Rows, DbError>;
}

// Only types in this crate implement Sealed (and thus DatabaseDriver)
pub struct PostgresDriver { /* ... */ }
impl private::Sealed for PostgresDriver {}
impl DatabaseDriver for PostgresDriver { /* ... */ }

// External code can USE DatabaseDriver but not implement it
```

## Non-Exhaustive Enums

Allow adding variants without breaking downstream:

```rust
#[non_exhaustive]
pub enum Event {
    Click { x: i32, y: i32 },
    KeyPress(char),
    Resize { width: u32, height: u32 },
    // Future variants won't break the API
}

// External match MUST have _ arm:
match event {
    Event::Click { x, y } => handle_click(x, y),
    Event::KeyPress(c) => handle_key(c),
    Event::Resize { width, height } => handle_resize(width, height),
    _ => {} // Required: future variants
}
```

## repr(transparent)

For newtypes that need identical memory layout to the wrapped type (FFI, safety):

```rust
#[repr(transparent)]
pub struct NonZeroU32(u32);

// Same size and alignment as u32 — safe to transmute
// Useful for: FFI safety, optimization, zero-overhead wrappers
```

## Pattern Selection Guide

| Need | Pattern |
|------|---------|
| Prevent ID swaps | Newtype |
| Validate at boundary | Newtype + `new()` returning `Result` |
| Required builder fields | Typestate Builder |
| State machine (compile-time) | Typestate with PhantomData |
| State machine (runtime) | Enum with transitions |
| Prevent external trait impl | Sealed Trait |
| Future-proof enum | `#[non_exhaustive]` |
| Generic type info, no data | PhantomData |

## References

- [Rust API Guidelines — Type Safety](https://rust-lang.github.io/api-guidelines/type-safety.html)
- [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
- [The Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/)
