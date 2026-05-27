# Typestate Pattern

Typestate encodes valid state transitions in types so invalid operations do not compile.

## Basic Pattern

```rust
pub struct Disconnected;
pub struct Connected;

pub struct Socket<State> {
    stream: Option<TcpStream>,
    _state: std::marker::PhantomData<State>,
}

impl Socket<Disconnected> {
    pub fn connect(addr: SocketAddr) -> Result<Socket<Connected>, Error> {
        let stream = TcpStream::connect(addr)?;
        Ok(Socket { stream: Some(stream), _state: std::marker::PhantomData })
    }
}

impl Socket<Connected> {
    pub fn send(&mut self, bytes: &[u8]) -> Result<(), Error> {
        self.stream.as_mut().unwrap().write_all(bytes)?;
        Ok(())
    }
}
```

`send` is unavailable before connection.

## Builders

Typestate builders enforce required fields at compile time.

```rust
pub struct Missing;
pub struct Present<T>(T);

pub struct RequestBuilder<Url> {
    url: Url,
}
```

## Tradeoffs

Pros:

- invalid states are unrepresentable;
- fewer runtime checks;
- documentation appears in type signatures.

Cons:

- more generic types;
- harder type errors;
- can overcomplicate simple workflows.

## Use When

Use typestate for security-sensitive protocols, multi-step builders, resource lifecycle, and APIs where misuse is common or costly.
