# Dioxus Fullstack, Hydration & Server Side Rendering

Dioxus Fullstack blends client-side UI rendering with server functions to construct isomorphic web applications.

## 1. Server Side Rendering (SSR) & Client Hydration

With SSR, the server renders the component tree to pure HTML, sends it to the browser, and the browser compiles the WASM package to attach event listeners (hydration).

```rust
// main.rs - Isomorphic execution
use dioxus::prelude::*;

fn main() {
    #[cfg(feature = "server")]
    {
        // Server bootstrap code
    }
    #[cfg(not(feature = "server"))]
    {
        // Client startup
        dioxus::launch(App);
    }
}
```

## 2. Server Functions: Transparent RPC

Server functions let components call server-side modules seamlessly without manual endpoint wiring.

```rust
use dioxus::prelude::*;
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
pub struct DbStats {
    pub rows: u64,
}

// Annotation directs compiler to compile this ONLY for the server target
// and sets up an RPC endpoint on the client WASM bundle
#[server(GetDatabaseStats)]
pub async fn get_database_stats() -> Result<DbStats, ServerFnError> {
    // Run SQL query directly
    Ok(DbStats { rows: 42 })
}

#[component]
fn Dashboard() -> Element {
    let stats = use_resource(|| get_database_stats());

    match &*stats.read_unchecked() {
        None => rsx! { "Fetching DB details..." },
        Some(Err(e)) => rsx! { "Error: {e}" },
        Some(Ok(data)) => rsx! {
            div { "Database Rows: {data.rows}" }
        }
    }
}
```

## 3. SSR Output Strategies

| Strategy | When to use | Trade-off |
|----------|------------|-----------|
| **Full SSR** | First paint priority, minimal JS | No interactivity until hydration |
| **SSR + hydration** | Balanced approach (Dioxus default) | Double render cost on client |
| **SSR with streaming** | Long pages, slow data | Progressive rendering, complex error handling |
| **Static generation** | Blog, marketing pages | No server needed, stale content |

## 4. Streaming SSR Patterns

```rust
use dioxus_fullstack::prelude::*;
use dioxus::prelude::*;

#[component]
fn SlowData(id: String) -> Element {
    // Data fetches on the server stream as they complete
    let data = use_resource(move || fetch_entity(id));

    rsx! {
        div {
            // This part renders immediately
            h2 { "Loading entity..." }
            // This part streams in when data resolves
            match data() {
                Some(Ok(entity)) => rsx! { entity.render() },
                Some(Err(e)) => rsx! { "Failed to load: {e}" },
                None => rsx! { p { "..." } },
            }
        }
    }
}
```

## 5. State Hydration Pitfalls

```rust
// Bad: client re-fetches what the server already loaded
#[component]
fn Profile() -> Element {
    let user = use_resource(fetch_user); // re-fetches on hydration
    // ...
}

// Good: share state across the boundary
#[component]
fn Profile() -> Element {
    // use_server_future sends serialized state to client
    let user = use_server_future(|| async {
        fetch_user().await
    })?;
    // Client sees pre-loaded data, no extra fetch
    // ...
}
```

## 6. Asset Management

- `dioxus-cli` bundles WASM, CSS, and assets automatically.
- Use `asset!()` macro for versioned asset paths in SSR HTML.
- Enable gzip/brotli compression on your reverse proxy for WASM payloads.
- Split large WASM binaries: defer non-critical components and lazy-load modules.

## 7. CORS and Cookie Forwarding

When server functions run on the server but call external APIs:

```rust
#[server(ProxyApi)]
pub async fn proxy_request() -> Result<ApiResponse, ServerFnError> {
    // Server-side code has access to the original request headers
    // through Dioxus context — forward auth cookies:
    let req = extract_request_data();
    let client = reqwest::Client::new();
    let resp = client
        .get("https://internal-api/data")
        .header("Cookie", req.cookie_header)
        .send()
        .await?;
    Ok(resp.json().await?)
}
```

## 8. Choosing a Deployment Model

| Model | Bundle | Server | Best for |
|-------|--------|--------|----------|
| Client-only WASM | All WASM | Static file host | Apps with minimal server needs |
| SSR with hydration | WASM + HTML | Rust server | SEO, first-paint sensitive |
| Full SSR (no WASM) | HTML only | Rust server | Content sites, low-interaction |
| Static generation | Pre-rendered HTML | None | Blogs, docs |
