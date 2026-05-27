---
name: rust-networking
description: |
  Rust networking and protocol development. Use when:
  - Working with TCP/UDP sockets, listeners, or raw socket I/O
  - Implementing TLS/SSL with rustls or native-tls
  - Building WebSocket clients or servers (tokio-tungstenite, tungstenite)
  - Using HTTP clients beyond reqwest (hyper, ureq, h3, isahc)
  - Implementing custom protocols or binary framing
  - Working with DNS resolution, network interfaces, or socket options
  - Building proxies, tunnels, or load balancers in Rust
  - Handling network errors, timeouts, backpressure, or connection pooling
  Trigger for any low-to-mid level networking or protocol implementation.
---

# Rust Networking

Rust's networking ecosystem combines memory safety with zero-cost abstractions, making it ideal for protocol implementations, proxies, and networked services.

## Core Crates

| Crate | Purpose | Async support |
|-------|---------|---------------|
| `std::net` | TCP/UDP, DNS resolution | Blocking only |
| `tokio::net` | TCP/UDP, Unix sockets, signal handling | Tokio |
| `async-std::net` | TCP/UDP | async-std |
| `mio` | Low-level I/O multiplexing (epoll/kqueue/IOCP) | Custom event loops |
| `socket2` | Socket creation, options, raw socket access | Platform-agnostic |

## TCP Server

### Blocking

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{BufRead, BufReader, Write};
use std::thread;

fn handle_client(mut stream: TcpStream) {
    let reader = BufReader::new(stream.try_clone().unwrap());
    for line in reader.lines().flatten() {
        let response = process_request(&line);
        writeln!(stream, "{response}").unwrap();
    }
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("0.0.0.0:8080")?;
    for stream in listener.incoming() {
        match stream {
            Ok(stream) => { thread::spawn(|| handle_client(stream)); }
            Err(e) => eprintln!("accept error: {e}"),
        }
    }
    Ok(())
}
```

### Async (Tokio)

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};

async fn handle_client(mut stream: TcpStream) {
    let (reader, mut writer) = stream.split();
    let mut lines = BufReader::new(reader).lines();
    while let Ok(Some(line)) = lines.next_line().await {
        let response = process_request(&line).await;
        if writer.write_all(response.as_bytes()).await.is_err() {
            break;
        }
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("0.0.0.0:8080").await?;
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(handle_client(stream));
    }
}
```

## TCP Client

```rust
use tokio::io::{AsyncWriteExt, BufReader, AsyncBufReadExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;
    stream.write_all(b"Hello server\n").await?;

    let mut reader = BufReader::new(&stream);
    let mut response = String::new();
    reader.read_line(&mut response).await?;
    println!("Server says: {response}");

    Ok(())
}
```

### Connecting with Timeout

```rust
use tokio::time::{timeout, Duration};

match timeout(Duration::from_secs(5), TcpStream::connect("10.0.0.1:8080")).await {
    Ok(Ok(stream)) => { /* connected */ }
    Ok(Err(e)) => eprintln!("connection error: {e}"),
    Err(_) => eprintln!("timeout connecting to 10.0.0.1:8080"),
}
```

## UDP

```rust
use tokio::net::UdpSocket;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let socket = UdpSocket::bind("0.0.0.0:34254").await?;
    socket.connect("127.0.0.1:8080").await?; // connect = filter sends/recvs

    socket.send(b"hello").await?;
    let mut buf = [0u8; 1024];
    let n = socket.recv(&mut buf).await?;
    println!("Received: {}", String::from_utf8_lossy(&buf[..n]));

    Ok(())
}
```

### UDP Broadcast Receiver

```rust
let socket = UdpSocket::bind("0.0.0.0:34254").await?;
socket.set_broadcast(true)?;

let mut buf = [0u8; 65535];
loop {
    let (n, src) = socket.recv_from(&mut buf).await?;
    println!("Received {} bytes from {src}", n);
}
```

## Socket Options

```rust
use socket2::{Socket, Domain, Type, Protocol};
use std::net::{TcpListener, TcpStream};

fn configure_socket(addr: &str) -> std::io::Result<TcpListener> {
    let socket = Socket::new(Domain::IPV4, Type::STREAM, Some(Protocol::TCP))?;

    // Essential server options
    socket.set_reuse_address(true)?;
    socket.set_reuse_port(true)?;      // SO_REUSEPORT — may not be available on all platforms
    socket.set_keepalive(true)?;
    socket.set_linger(Some(std::time::Duration::from_secs(10)))?;
    socket.set_nodelay(true)?;         // Disable Nagle's algorithm
    socket.set_send_buffer_size(64 * 1024)?;
    socket.set_recv_buffer_size(64 * 1024)?;

    socket.set_nonblocking(true)?;
    let address: std::net::SocketAddr = addr.parse().unwrap();
    socket.bind(&address.into())?;
    socket.listen(1024)?;  // backlog

    // Convert back to std TcpListener
    let listener: TcpListener = socket.into();
    Ok(listener)
}
```

## Binary Framing

When building custom protocols, frame messages to delimit boundaries.

```rust
use bytes::{BytesMut, BufMut, Bytes};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

/// Length-prefixed framing: [4-byte len][payload]
async fn read_frame(stream: &mut (impl AsyncReadExt + Unpin)) -> std::io::Result<Bytes> {
    let mut len_buf = [0u8; 4];
    stream.read_exact(&mut len_buf).await?;
    let len = u32::from_be_bytes(len_buf) as usize;

    let mut payload = vec![0u8; len];
    stream.read_exact(&mut payload).await?;
    Ok(Bytes::from(payload))
}

async fn write_frame(stream: &mut (impl AsyncWriteExt + Unpin), payload: &[u8]) -> std::io::Result<()> {
    let len = payload.len() as u32;
    stream.write_all(&len.to_be_bytes()).await?;
    stream.write_all(payload).await?;
    Ok(())
}
```

## Connection Pooling

```rust
use deadpool::managed::{self, RecycleResult, Manager};
use tokio::net::TcpStream;

struct TcpConnectionManager;

impl Manager for TcpConnectionManager {
    type Type = TcpStream;
    type Error = std::io::Error;

    async fn create(&self) -> Result<TcpStream, Self::Error> {
        TcpStream::connect("127.0.0.1:8080").await
    }

    async fn recycle(&self, conn: &mut TcpStream) -> RecycleResult<Self::Error> {
        // Check if connection is still alive
        Ok(())
    }
}

type Pool = managed::Pool<TcpConnectionManager>;

async fn use_pool(pool: &Pool) {
    let mut conn = pool.get().await.unwrap();
    // use conn...
} // returned to pool on drop
```

## Network Errors and Resilience

```rust
use std::io;
use tokio::time::{sleep, Duration};

/// Exponential backoff with jitter
async fn connect_with_retry(addr: &str, max_retries: u32) -> io::Result<TcpStream> {
    let mut delay = Duration::from_millis(100);

    for attempt in 0..max_retries {
        match TcpStream::connect(addr).await {
            Ok(stream) => return Ok(stream),
            Err(e) if attempt == max_retries - 1 => return Err(e),
            Err(_) => {
                // Add jitter: randomize delay ±50%
                let jitter = rand::random::<f64>() * 0.5 + 0.75;
                sleep(delay.mul_f64(jitter)).await;
                delay *= 2; // exponential backoff
                if delay > Duration::from_secs(30) {
                    delay = Duration::from_secs(30); // cap
                }
            }
        }
    }
    unreachable!()
}
```

## Anti-Patterns

```rust
// Bad: blocking the event loop with synchronous I/O
#[tokio::main]
async fn main() {
    let listener = std::net::TcpListener::bind("0.0.0.0:8080").unwrap(); // blocks!
    for stream in listener.incoming() { /* ... */ }
}

// Good: use tokio::net types
let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

// Bad: no timeout on connect/read
TcpStream::connect("slow-host:8080").await?; // could hang indefinitely

// Good: always timeout network I/O
tokio::time::timeout(Duration::from_secs(10), TcpStream::connect(addr)).await??;
```

```rust
// Bad: unbounded read buffer
let mut buf = Vec::new();
stream.read_to_end(&mut buf).await?; // OOM if peer sends forever

// Good: bounded read
let mut buf = vec![0u8; MAX_FRAME_SIZE];
let n = stream.read(&mut buf).await?;
```

## Quick Navigation

- [references/tcp_udp.md](references/tcp_udp.md) — TCP/UDP sockets, listeners, streaming
- [references/tls.md](references/tls.md) — TLS/SSL with rustls and native-tls
- [references/websockets.md](references/websockets.md) — WebSocket client/server with tokio-tungstenite
- [references/http_clients.md](references/http_clients.md) — HTTP clients (hyper, ureq, h3)
- [references/grpc.md](references/grpc.md) — gRPC with tonic

## Review Checklist

- Socket timeouts configured on all network I/O
- Read/write buffers are bounded, not unbounded
- TLS certificate validation is enabled (no `dangerous_accept_invalid_certs`)
- Connection pools have max size and recycle policy
- Graceful shutdown drains in-flight connections
- Backpressure propagates through the protocol layer

## References

- [tokio::net docs](https://docs.rs/tokio/latest/tokio/net/index.html)
- [socket2 docs](https://docs.rs/socket2)
- [rustls book](https://rustls.github.io/)
- [tokio-tungstenite](https://docs.rs/tokio-tungstenite)
- [hyper guide](https://hyper.rs/guides/)
- [tonic docs](https://docs.rs/tonic)
