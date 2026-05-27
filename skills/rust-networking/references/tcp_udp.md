# TCP/UDP Sockets

Low-level socket programming in Rust for custom protocols, proxies, and network services.

## TCP Frame Processing

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt, AsyncBufReadExt, BufReader};
use bytes::BytesMut;

/// Read until a delimiter byte is found
async fn read_until_delimiter(
    stream: &mut (impl AsyncReadExt + Unpin),
    delimiter: u8,
    buf: &mut BytesMut,
) -> std::io::Result<usize> {
    loop {
        if let Some(pos) = buf.iter().position(|&b| b == delimiter) {
            let end = pos + 1;
            return Ok(end);
        }
        if buf.len() >= MAX_MESSAGE_SIZE {
            return Err(std::io::Error::new(std::io::ErrorKind::InvalidData, "message too large"));
        }
        let mut tmp = [0u8; 1024];
        let n = stream.read(&mut tmp).await?;
        if n == 0 {
            return Err(std::io::Error::new(std::io::ErrorKind::UnexpectedEof, "connection closed"));
        }
        buf.extend_from_slice(&tmp[..n]);
    }
}
```

## Line-Based Protocol (Redis-style)

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpStream;

async fn handle_redis_like(mut stream: TcpStream) {
    let (reader, mut writer) = stream.split();
    let mut lines = BufReader::new(reader).lines();

    while let Ok(Some(line)) = lines.next_line().await {
        match line.to_uppercase().as_str() {
            "PING" => {
                let _ = writer.write_all(b"+PONG\r\n").await;
            }
            cmd if cmd.starts_with("ECHO ") => {
                let msg = &line[5..];
                let _ = writer.write_all(format!("${}\r\n{msg}\r\n", msg.len()).as_bytes()).await;
            }
            _ => {
                let _ = writer.write_all(b"-ERR unknown command\r\n").await;
            }
        }
    }
}
```

## Nagle's Algorithm and TCP_NODELAY

```rust
// Disable Nagle's for low-latency protocols (HTTP/2, WebSocket, real-time)
stream.set_nodelay(true)?;

// Default (Nagle enabled): coalesces small writes — good for bulk transfers
// Disabled: sends immediately — good for interactive/real-time protocols

// With nodelay off, this:
stream.write_all(b"H").await?;
stream.write_all(b"e").await?;
stream.write_all(b"l").await?;
// May be coalesced into a single TCP segment
```

## TCP Keep-Alive

```rust
use std::time::Duration;

// Enable keep-alive with custom parameters (platform-dependent)
// On Linux via socket2:
let socket: socket2::Socket = stream.into();
socket.set_tcp_keepalive(true)?;
socket.set_tcp_keepalive_time(Duration::from_secs(30))?;     // idle before probe
socket.set_tcp_keepalive_interval(Duration::from_secs(10))?;   // between probes
socket.set_tcp_keepalive_retries(5_usize)?;                    // probes before close
let stream: TcpStream = socket.into();
```

## UDP Connection Semantics

```rust
use tokio::net::UdpSocket;

// "Connected" UDP: restricts to single peer
let socket = UdpSocket::bind("0.0.0.0:0").await?;
socket.connect("8.8.8.8:53").await?; // connect = set default destination
socket.send(dns_query).await?;       // always goes to 8.8.8.8:53
let n = socket.recv(&mut buf).await?; // only receives from 8.8.8.8:53

// Unconnected UDP: can send/recv from any peer
let socket = UdpSocket::bind("0.0.0.0:34254").await?;
socket.send_to(data, "peer1:8080").await?;
socket.send_to(other_data, "peer2:8080").await?;
let (n, src) = socket.recv_from(&mut buf).await?;
```

## UDP MTU and Fragmentation

```rust
// IPv4: minimum MTU is 576 bytes, typical is 1500
// UDP header: 8 bytes; IP header: 20 bytes
// Safe payload size: 1500 - 20 - 8 = 1472 bytes (typical Ethernet)
// Guaranteed to not fragment: 576 - 20 - 8 = 548 bytes

// To avoid IP fragmentation, keep UDP payloads ≤ 1472 for Ethernet
// or ≤ 548 for guaranteed delivery across all networks
let MAX_SAFE_UDP_PAYLOAD = 1472;

// On Linux, detect fragmentation with:
socket.set_msg_mtu_discovery(true)?; // sets IP_MTU_DISCOVER
// Then recv() returns EMSGSIZE if message exceeds path MTU
```

## Listening on Multiple Addresses

```rust
use tokio::net::TcpListener;
use tokio::try_join;

async fn serve_dual() -> std::io::Result<()> {
    let v4 = TcpListener::bind("0.0.0.0:8080").await?;
    let v6 = TcpListener::bind("[::]:8080").await?;

    loop {
        let (stream_v4, _) = v4.accept().await?;
        tokio::spawn(handle(stream_v4));
        let (stream_v6, _) = v6.accept().await?;
        tokio::spawn(handle(stream_v6));
    }
}
```

## Backpressure in TCP

```rust
use tokio::sync::mpsc;
use tokio::io::AsyncWriteExt;

async fn client_writer(
    mut stream: TcpStream,
    mut rx: mpsc::Receiver<Vec<u8>>,
) {
    while let Some(data) = rx.recv().await {
        // If the kernel send buffer is full, this await yields control
        // — natural backpressure propagates to the producer
        if stream.write_all(&data).await.is_err() {
            break; // connection lost
        }
    }
}

// Producer side backpressure
async fn producer(tx: mpsc::Sender<Vec<u8>>) {
    loop {
        let data = generate_data().await;
        // If channel is full (consumer is slow), await here = backpressure
        if tx.send(data).await.is_err() {
            break; // consumer dropped
        }
    }
}
```

## TCP Half-Close

```rust
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

async fn half_close_example(mut stream: TcpStream) {
    // Send request
    stream.write_all(request).await.unwrap();
    // Signal we're done sending — peer reads EOF on recv
    stream.shutdown().await.unwrap(); // SHUT_WR

    // Can still read the response
    let mut response = String::new();
    stream.read_to_string(&mut response).await.unwrap();
}
```

## Anti-Patterns

- Reading without a buffer limit.
- No connect timeout — network I/O can hang indefinitely.
- Forgetting `set_nodelay(true)` for interactive protocols.
- Blocking in an async context — use `tokio::net`, not `std::net`.
- Ignoring `WouldBlock` errors in non-blocking mode — retry via readiness.
