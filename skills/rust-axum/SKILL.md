---
name: rust-axum
description: |
  Axum web framework for building HTTP APIs in Rust. Use this skill when:
  - User builds a REST API or web server with Axum
  - User asks about Axum routing, handlers, extractors, or middleware
  - User works with Axum state management (AppState, Extension)
  - User integrates Axum with Tower middleware, tracing, or CORS
  - User handles request/response bodies (JSON, form, multipart)
  - User implements authentication middleware or request validation
  - User asks about WebSocket handlers in Axum
  - User configures graceful shutdown for an Axum server
  - User asks about error handling in Axum (IntoResponse, AppError)
  Trigger for any Axum HTTP API development question.
---



## Quick Navigation

- [references/routing_extractors.md](references/routing_extractors.md)
- [references/middleware_state.md](references/middleware_state.md)

# Axum Web Framework

Axum is the Tower-native, Tokio-first HTTP framework. It composes with any Tower middleware and has zero magic — handlers are plain async functions.

## Quick Reference

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace", "compression-gzip"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

## Minimal Server

```rust
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(root))
        .route("/health", get(health));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}

async fn root() -> &'static str { "Hello, world!" }
async fn health() -> axum::http::StatusCode { axum::http::StatusCode::OK }
```

## Routing

```rust
use axum::{
    routing::{delete, get, post, put},
    Router,
};

let app = Router::new()
    // RESTful CRUD
    .route("/users",          get(list_users).post(create_user))
    .route("/users/:id",      get(get_user).put(update_user).delete(delete_user))
    // Nested routers
    .nest("/api/v1", api_v1_router())
    // Static files (tower-http)
    .nest_service("/static", tower_http::services::ServeDir::new("public"));

fn api_v1_router() -> Router<AppState> {
    Router::new()
        .route("/posts", get(list_posts))
}
```

## State Management

```rust
use axum::{extract::State, Router};
use std::sync::Arc;

// AppState: wrap in Arc for cheap cloning across handlers
#[derive(Clone)]
struct AppState {
    db: Arc<DbPool>,
    config: Arc<Config>,
}

async fn get_user(
    State(state): State<Arc<AppState>>,
    axum::extract::Path(id): axum::extract::Path<u64>,
) -> Result<axum::Json<User>, AppError> {
    let user = state.db.find_user(id).await?;
    Ok(axum::Json(user))
}

let state = Arc::new(AppState { db, config });
let app = Router::new()
    .route("/users/:id", get(get_user))
    .with_state(state);
```

## Extractors

Extractors are types that implement `FromRequest` or `FromRequestParts`. They run before the handler.

```rust
use axum::{
    extract::{Json, Path, Query, State},
    http::HeaderMap,
};
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination {
    page: Option<u32>,
    per_page: Option<u32>,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

// Multiple extractors in one handler
async fn list_users(
    State(db): State<Arc<DbPool>>,
    Query(pagination): Query<Pagination>,
    headers: HeaderMap,
) -> axum::Json<Vec<User>> {
    let page = pagination.page.unwrap_or(1);
    let per_page = pagination.per_page.unwrap_or(20);
    axum::Json(db.list_users(page, per_page).await.unwrap())
}

// JSON body — must be last extractor (consumes body)
async fn create_user(
    State(db): State<Arc<DbPool>>,
    Json(payload): Json<CreateUser>,
) -> Result<(axum::http::StatusCode, axum::Json<User>), AppError> {
    let user = db.create_user(&payload.name, &payload.email).await?;
    Ok((axum::http::StatusCode::CREATED, axum::Json(user)))
}
```

## Error Handling

The idiomatic pattern: define `AppError`, implement `IntoResponse`.

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

#[derive(Debug)]
pub enum AppError {
    NotFound(String),
    Unauthorized,
    InvalidInput(String),
    Internal(anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "Unauthorized".into()),
            AppError::InvalidInput(msg) => (StatusCode::UNPROCESSABLE_ENTITY, msg),
            AppError::Internal(e) => {
                tracing::error!("Internal error: {e:#}");
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error".into())
            }
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}

// Convert from anyhow::Error automatically
impl From<anyhow::Error> for AppError {
    fn from(err: anyhow::Error) -> Self {
        AppError::Internal(err)
    }
}

// Now handlers return Result<T, AppError>
async fn get_post(Path(id): Path<u64>) -> Result<Json<Post>, AppError> {
    let post = db.find_post(id).await
        .map_err(AppError::Internal)?
        .ok_or_else(|| AppError::NotFound(format!("Post {id} not found")))?;
    Ok(Json(post))
}
```

## Middleware

Axum uses Tower middleware. Prefer `tower_http` for common needs:

```rust
use tower_http::{
    cors::{Any, CorsLayer},
    trace::TraceLayer,
    compression::CompressionLayer,
    request_id::{MakeRequestUuid, SetRequestIdLayer, PropagateRequestIdLayer},
};
use axum::http::header;

let app = Router::new()
    .route("/", get(handler))
    // Request tracing (integrates with tracing crate)
    .layer(TraceLayer::new_for_http())
    // Response compression
    .layer(CompressionLayer::new())
    // CORS
    .layer(
        CorsLayer::new()
            .allow_origin(Any)
            .allow_methods(Any)
            .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION]),
    )
    // Request IDs
    .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))
    .layer(PropagateRequestIdLayer::x_request_id());
```

### Custom Middleware

```rust
use axum::{middleware, extract::Request, response::Response};

async fn auth_middleware(
    State(state): State<Arc<AppState>>,
    mut req: Request,
    next: middleware::Next,
) -> Result<Response, AppError> {
    let token = req
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(AppError::Unauthorized)?;

    let user = state.auth.validate_token(token).await
        .map_err(|_| AppError::Unauthorized)?;

    // Pass user to handlers via extensions
    req.extensions_mut().insert(user);
    Ok(next.run(req).await)
}

// Apply to specific routes
let protected = Router::new()
    .route("/profile", get(get_profile))
    .layer(middleware::from_fn_with_state(state.clone(), auth_middleware));
```

## WebSockets

```rust
use axum::extract::ws::{Message, WebSocket, WebSocketUpgrade};

async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    loop {
        tokio::select! {
            Some(Ok(msg)) = socket.recv() => {
                match msg {
                    Message::Text(text) => {
                        if socket.send(Message::Text(format!("echo: {text}"))).await.is_err() {
                            break;
                        }
                    }
                    Message::Close(_) => break,
                    _ => {}
                }
            }
            else => break,
        }
    }
}
```

## Graceful Shutdown

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    let app = build_router();
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }

    println!("Shutdown signal received, shutting down...");
}
```

## Project Structure

```
src/
├── main.rs           # Entry point: build app, bind listener
├── lib.rs            # App builder fn (for integration tests)
├── router.rs         # Route definitions
├── state.rs          # AppState definition
├── error.rs          # AppError + IntoResponse
├── handlers/
│   ├── users.rs
│   └── posts.rs
├── middleware/
│   ├── auth.rs
│   └── logging.rs
└── models/           # Domain types
    ├── user.rs
    └── post.rs
```

## Common Gotchas

```rust
// 1. State must be Clone — wrap expensive resources in Arc
#[derive(Clone)]  // Required for State<>
struct AppState {
    db: Arc<Pool<Postgres>>, // Arc makes clone cheap
}

// 2. Body extractors (Json, Bytes, String) must be LAST
async fn handler(
    State(s): State<AppState>, // ✓ non-consuming
    Path(id): Path<u64>,       // ✓ non-consuming
    Json(body): Json<Input>,   // ✓ last: consumes body
) {}

// 3. Handler return types must implement IntoResponse
// (axum::Json<T>, StatusCode, (StatusCode, Json<T>), etc.)

// 4. For typed path segments use Path<(T1, T2)>
async fn handler(Path((user_id, post_id)): Path<(u64, u64)>) {}
// Route: "/users/:user_id/posts/:post_id"
```

## Anti-Patterns

```rust
// Bad: body extractor before other extractors; later extractors cannot read request parts.
async fn bad(Json(body): Json<Input>, Path(id): Path<u64>) {}

// Good: body-consuming extractor last.
async fn good(Path(id): Path<u64>, Json(body): Json<Input>) {}
```

```rust
// Bad: cloning a pool wrapper manually per request.
struct AppState { db: Pool<Postgres> }

// Good: cheap Clone state; expensive resources are internally shared or wrapped in Arc.
#[derive(Clone)]
struct AppState { db: Arc<Pool<Postgres>> }
```

## Production Checklist

- Put body extractors last in handler signatures.
- Keep `AppState` cheap to clone; use `Arc` for expensive shared resources.
- Convert domain errors through one `IntoResponse` path.
- Add `TraceLayer`, request IDs, timeouts, and CORS deliberately.
- Exercise routes through `Router::oneshot` or real listener integration tests.
- Verify graceful shutdown drains in-flight requests.

## References

- [Axum docs](https://docs.rs/axum)
- [Axum examples](https://github.com/tokio-rs/axum/tree/main/examples)
- [tower-http](https://docs.rs/tower-http)
- [Tokio blog: Axum overview](https://tokio.rs/blog/2021-07-announcing-axum)
