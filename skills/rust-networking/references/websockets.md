# WebSocket Clients & Servers

Real-time bidirectional communication with tokio-tungstenite.

## Dependencies

```toml
[dependencies]
tokio-tungstenite = { version = "0.24", features = ["native-tls"] }
# or for rustls:
tokio-tungstenite = { version = "0.24", features = ["rustls-tls-webpki-roots"] }

futures-util = "0.3"  # for StreamExt/SinkExt
```

## WebSocket Server

```rust
use tokio::net::TcpListener;
use tokio_tungstenite::accept_async;
use tokio_tungstenite::tungstenite::Message;
use futures_util::{SinkExt, StreamExt};

async fn ws_server() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:9001").await?;

    while let Ok((stream, _)) = listener.accept().await {
        tokio::spawn(handle_connection(stream));
    }
    Ok(())
}

async fn handle_connection(stream: tokio::net::TcpStream) {
    let ws_stream = accept_async(stream).await.unwrap();
    let (mut sender, mut receiver) = ws_stream.split();

    // Echo server
    while let Some(Ok(msg)) = receiver.next().await {
        if msg.is_text() || msg.is_binary() {
            if sender.send(msg).await.is_err() {
                break; // client disconnected
            }
        } else if msg.is_close() {
            break;
        }
    }
}
```

## WebSocket Client

```rust
use tokio_tungstenite::connect_async;
use tokio_tungstenite::tungstenite::Message;
use futures_util::{SinkExt, StreamExt};

async fn ws_client() -> Result<(), Box<dyn std::error::Error>> {
    let url = url::Url::parse("wss://echo.websocket.org")?;
    let (ws_stream, _) = connect_async(url).await?;

    let (mut sender, mut receiver) = ws_stream.split();

    // Send a message
    sender.send(Message::Text("hello".into())).await?;

    // Receive response
    if let Some(Ok(msg)) = receiver.next().await {
        println!("Received: {msg}");
    }

    sender.send(Message::Close(None)).await?;
    Ok(())
}
```

## Ping/Pong

```rust
use tokio_tungstenite::tungstenite::Message;
use tokio::time::{interval, Duration};

// Server-side: tungstenite handles ping/pong automatically
// by sending pong in response to ping, and closing on no response

// Client-side custom keepalive
async fn keepalive(sender: &mut (impl SinkExt<Message> + Unpin)) {
    let mut ticker = interval(Duration::from_secs(15));
    loop {
        ticker.tick().await;
        if sender.send(Message::Ping(vec![])).await.is_err() {
            break; // connection dead
        }
    }
}
```

## Broadcasting to Multiple Clients (Pub/Sub)

```rust
use tokio::sync::broadcast;
use tokio_tungstenite::accept_async;
use tokio_tungstenite::tungstenite::Message;
use futures_util::{SinkExt, StreamExt};
use std::sync::Arc;

async fn broadcast_server() {
    let (tx, _) = broadcast::channel::<String>(100);
    let listener = TcpListener::bind("0.0.0.0:9001").await.unwrap();

    loop {
        let (stream, _) = listener.accept().await.unwrap();
        let tx = tx.clone();
        let rx = tx.subscribe();

        tokio::spawn(handle_client(stream, tx.clone(), rx));
    }
}

async fn handle_client(
    stream: tokio::net::TcpStream,
    tx: broadcast::Sender<String>,
    mut rx: broadcast::Receiver<String>,
) {
    let ws_stream = accept_async(stream).await.unwrap();
    let (mut sender, mut receiver) = ws_stream.split();

    // Forward broadcast messages to this client
    let send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(Message::Text(msg)).await.is_err() {
                break;
            }
        }
    });

    // Forward client messages to broadcast
    while let Some(Ok(msg)) = receiver.next().await {
        if let Message::Text(text) = msg {
            let _ = tx.send(text);
        }
    }

    send_task.abort();
}
```

## WebSocket with TLS (WSS)

```rust
use tokio_tungstenite::connect_async;
use tokio_native_tls::TlsConnector;
use native_tls::TlsConnector as NativeTlsConnector;

// Client side with native-tls
async fn wss_client() -> Result<(), Box<dyn std::error::Error>> {
    let url = url::Url::parse("wss://example.com/ws")?;
    let (ws_stream, _) = connect_async(url).await?;
    // Use ws_stream as normal
    Ok(())
}

// Server side with TLS — wrap TcpListener with TlsAcceptor first,
// then accept_async on the TlsStream
```

## Connection State Machine

```rust
#[derive(Debug, PartialEq)]
enum ConnectionState {
    Connecting,
    Open,
    Closing,
    Closed,
}

struct WsConnection {
    state: ConnectionState,
}

impl WsConnection {
    fn on_open(&mut self) {
        self.state = ConnectionState::Open;
    }

    fn on_message(&self, msg: &Message) {
        match self.state {
            ConnectionState::Open => { /* process */ }
            ConnectionState::Closing => {
                // Only accept close frames
                if !msg.is_close() {
                    // ignore or error
                }
            }
            _ => { /* ignore */ }
        }
    }

    fn on_close(&mut self) {
        self.state = ConnectionState::Closed;
    }
}
```

## Configuring WebSocket

```rust
use tokio_tungstenite::tungstenite::protocol::WebSocketConfig;

let config = WebSocketConfig {
    max_message_size: Some(1024 * 1024),       // 1 MB max message
    max_frame_size: Some(65536),               // 64 KB max frame
    ..Default::default()
};

// Pass config to accept or connect
let ws_stream = accept_async_with_config(stream, Some(config)).await?;
// or
let (ws_stream, _) = connect_async_with_config(url, Some(config)).await?;
```

## Anti-Patterns

```rust
// Bad: unbounded message memory
// ws_stream.next() in a loop without max_message_size config

// Bad: blocking the WebSocket read loop with slow async work
while let Some(Ok(msg)) = receiver.next().await {
    process_slow(msg).await; // blocks reading — buffer fills, connection drops
}

// Good: spawn a task for processing
while let Some(Ok(msg)) = receiver.next().await {
    let tx = sender.clone();
    tokio::spawn(async move {
        let response = process_slow(msg).await;
        let _ = tx.send(response).await;
    });
}
```

- Not handling pings/pongs for idle connections.
- Forgetting to limit `max_message_size` — OOM vector.
- Blocking the tokio runtime in the read loop.
- Using unbounded channels for message broadcasting.
