# Behavioral Patterns in Rust

Behavioral patterns organize how objects, functions, and state transitions communicate.

## Strategy via Traits

Use traits or generics to swap algorithms.

```rust
pub trait Compression {
    fn compress(&self, input: &[u8]) -> Vec<u8>;
}

pub struct Encoder<C> {
    compression: C,
}

impl<C: Compression> Encoder<C> {
    pub fn encode(&self, input: &[u8]) -> Vec<u8> {
        self.compression.compress(input)
    }
}
```

Use `Box<dyn Compression>` when the strategy is selected at runtime.

## State Machine

Represent states explicitly instead of scattering booleans.

```rust
pub enum ConnectionState {
    Disconnected,
    Connecting { attempt: u8 },
    Connected { session_id: String },
}
```

## Typestate

Use type parameters to make invalid transitions unrepresentable.

```rust
pub struct Draft;
pub struct Published;

pub struct Post<State> {
    title: String,
    body: String,
    _state: std::marker::PhantomData<State>,
}

impl Post<Draft> {
    pub fn publish(self) -> Post<Published> {
        Post { title: self.title, body: self.body, _state: std::marker::PhantomData }
    }
}
```

## Command Pattern

Represent actions as values for queues, undo logs, or retries.

```rust
pub enum Command {
    CreateUser { name: String },
    DisableUser { id: UserId },
}
```

## Observer with Channels

Use channels for event delivery across tasks.

```rust
let (tx, _) = tokio::sync::broadcast::channel(128);
let mut subscriber = tx.subscribe();

tx.send(Event::ConfigReloaded)?;
while let Ok(event) = subscriber.recv().await {
    handle(event).await;
}
```

## Visitor Alternatives

Rust often favors enums + pattern matching over OO visitors.

```rust
pub enum Expr {
    Number(i64),
    Add(Box<Expr>, Box<Expr>),
}

pub fn eval(expr: &Expr) -> i64 {
    match expr {
        Expr::Number(n) => *n,
        Expr::Add(a, b) => eval(a) + eval(b),
    }
}
```

## Anti-Patterns

Trait objects where enum dispatch is simpler, boolean flags that encode hidden state machines, and global mutable observers instead of explicit channels.
