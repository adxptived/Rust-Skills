# HTTP Clients

Rust HTTP client libraries for various use cases: from lightweight sync requests to full async HTTP/2 and HTTP/3.

## Client Library Comparison

| Crate | Async | HTTP/2 | HTTP/3 | Streaming | Binary size | Dependencies |
|-------|-------|--------|--------|-----------|-------------|--------------|
| `reqwest` | Yes (tokio) | Yes (native-tls) | No | Yes | ~2 MB | Many (hyper-based) |
| `ureq` | No (blocking) | No | No | Yes | ~500 KB | Minimal |
| `hyper` | Yes (tokio) | Yes | No | Yes | ~1.5 MB | Many (low-level) |
| `isahc` | Yes (curl) | Yes | Yes | Yes | ~3 MB | libcurl |
| `attohttpc` | No (blocking) | No | No | Limited | ~300 KB | Minimal |
| `h3` + `quinn` | Yes | No | Yes | Yes | ~4 MB | QUIC stack |
| `actix-web` Client | Yes (actix) | Yes | No | Yes | ~2 MB | actix-rt |

Use `reqwest` for general purpose, `ureq` for minimal dependencies, `hyper` for custom low-level control.

## Reqwest (Most Common)

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json", "rustls-tls"] }
```

### GET with JSON Response

```rust
use reqwest::Client;
use serde::Deserialize;

#[derive(Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

async fn fetch_user(client: &Client, user_id: u64) -> Result<User, reqwest::Error> {
    let user: User = client
        .get(format!("https://api.example.com/users/{user_id}"))
        .header("Accept", "application/json")
        .send()
        .await?
        .error_for_status()?
        .json()
        .await?;
    Ok(user)
}
```

### POST with JSON Body

```rust
#[derive(Serialize)]
struct CreateUser {
    name: String,
    email: String,
}

async fn create_user(client: &Client, data: &CreateUser) -> Result<User, reqwest::Error> {
    let user: User = client
        .post("https://api.example.com/users")
        .json(data)
        .send()
        .await?
        .error_for_status()?
        .json()
        .await?;
    Ok(user)
}
```

### Streaming Responses

```rust
use tokio::io::AsyncWriteExt;

async fn download_file(client: &Client, url: &str, path: &str) -> Result<(), reqwest::Error> {
    let response = client.get(url).send().await?;
    let mut file = tokio::fs::File::create(path).await.unwrap();

    let mut stream = response.bytes_stream();
    use futures_util::StreamExt;
    while let Some(chunk) = stream.next().await {
        let chunk = chunk?;
        file.write_all(&chunk).await.unwrap();
    }
    Ok(())
}
```

### Reusable Client Configuration

```rust
use std::time::Duration;

fn build_client() -> reqwest::Client {
    reqwest::Client::builder()
        .timeout(Duration::from_secs(30))
        .connect_timeout(Duration::from_secs(10))
        .user_agent("my-app/1.0")
        .default_headers({
            let mut headers = reqwest::header::HeaderMap::new();
            headers.insert("Accept", "application/json".parse().unwrap());
            headers
        })
        .pool_max_idle_per_host(10)    // max idle connections per host
        .pool_idle_timeout(Duration::from_secs(90))
        .http2_keep_alive_interval(Some(Duration::from_secs(30)))
        .build()
        .unwrap()
}
```

## Ureq (Lightweight, Blocking)

```toml
[dependencies]
ureq = { version = "3", features = ["json"] }
```

```rust
fn fetch_user_sync(user_id: u64) -> Result<User, ureq::Error> {
    let user: User = ureq::get(&format!("https://api.example.com/users/{user_id}"))
        .call()?
        .into_json()?;
    Ok(user)
}

// With custom agent and config
fn build_ureq_agent() -> ureq::Agent {
    ureq::AgentBuilder::new()
        .timeout_connect(Duration::from_secs(10))
        .timeout_read(Duration::from_secs(30))
        .timeout_write(Duration::from_secs(10))
        .user_agent("my-app/1.0")
        .build()
}
```

## Hyper (Low-Level, Full Control)

```rust
use hyper_util::client::legacy::Client;
use hyper::body::Bytes;
use hyper_util::rt::TokioExecutor;

async fn hyper_request() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::builder(TokioExecutor::new()).build_http();

    let req = hyper::Request::builder()
        .uri("http://httpbin.org/ip")
        .header("Accept", "application/json")
        .body(hyper::Body::empty())?;

    let resp = client.request(req).await?;
    let body = hyper::body::to_bytes(resp.into_body()).await?;
    println!("Response: {}", String::from_utf8_lossy(&body));

    Ok(())
}
```

## Retry and Resilience

```rust
use reqwest_middleware::{ClientBuilder, ClientMiddleware};
use reqwest_retry::{RetryTransientMiddleware, policies::ExponentialBackoff};
use reqwest::Client;

fn resilient_client() -> Client {
    let retry_policy = ExponentialBackoff::builder()
        .retry_bounds(Duration::from_secs(1), Duration::from_secs(60))
        .build_with_max_retries(3);

    ClientBuilder::new(Client::new())
        .with(RetryTransientMiddleware::new_with_policy(retry_policy))
        .build()
}

// Manual retry for non-transient errors
async fn fetch_with_retry(client: &Client, url: &str) -> Result<String, reqwest::Error> {
    let mut last_error = None;
    for _ in 0..3 {
        match client.get(url).send().await {
            Ok(resp) => return resp.text().await,
            Err(e) => last_error = Some(e),
        }
        tokio::time::sleep(Duration::from_millis(200)).await;
    }
    Err(last_error.unwrap())
}
```

## HTTP/3 with Quinn + H3

```rust
// Experimental — requires nightly or specific feature flags
// Cargo.toml:
//   h3 = { git = "https://github.com/hyperium/h3" }
//   quinn = "0.11"
//
// async fn h3_request() -> Result<(), Box<dyn std::error::Error>> {
//     let mut endpoint = quinn::Endpoint::client("[::]:0".parse()?)?;
//     let conn = endpoint.connect("example.com:443", "example.com")?.await?;
//     let mut quic = h3::client::Connection::new(h3_quinn::Connection::new(conn));
//     let (mut tx, mut rx) = quic.send_request(GET_HEADERS).await?;
//     // ...
// }
```

## Proxy Support

```rust
use reqwest::Proxy;

let client = Client::builder()
    .proxy(Proxy::http("http://proxy:8080")?)
    .proxy(Proxy::https("http://proxy:8080")?)
    .no_proxy("localhost,127.0.0.1,.internal")
    .build()?;
```

## Anti-Patterns

```rust
// Bad: creating a new Client for every request (costly — no connection reuse)
for url in urls {
    let resp = reqwest::get(url).await?; // creates new connection each time
}

// Good: reuse a single client
let client = Client::new();
for url in urls {
    let resp = client.get(url).send().await?; // reuses connections
}

// Bad: no timeout — request can hang forever
let resp = client.get(url).send().await?;

// Good: timeout configured on client or per-request
let resp = tokio::time::timeout(
    Duration::from_secs(30),
    client.get(url).send()
).await???;

// Bad: not checking for HTTP error status codes
let resp = client.get(url).send().await?;
// resp may be 404 or 500 — .error_for_status() converts to Err
```

- Ignoring `Content-Length` / chunked encoding in streaming responses.
- Not setting a User-Agent (some servers reject requests without one).
- Building a new TLS client config per-connection (session resumption is lost).
- Using blocking clients inside async contexts.
