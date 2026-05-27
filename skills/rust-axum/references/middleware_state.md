# Axum State & Middleware: Architectural Patterns

Axum is designed to run on top of Tower's middleware infrastructure, enabling powerful composition of cross-cutting concerns (authentication, logging, CORS, rate limiting).

## 1. Application State Architecture

State in Axum is shared across handlers using `Router::with_state`. The state type must be `Clone` and is typically wrapped in `Arc` if it contains connection pools, cache clients, or active resource managers.

```rust
use axum::{extract::State, routing::get, Router};
use std::sync::Arc;

#[derive(Clone)]
pub struct AppState {
    pub db: Arc<DbPool>,
    pub cache: Arc<CacheClient>,
}

pub fn app_router(state: AppState) -> Router {
    Router::new()
        .route("/data", get(fetch_data_handler))
        .with_state(state)
}

async fn fetch_data_handler(State(state): State<AppState>) -> &'static str {
    // Access state.db and state.cache here
    "Data fetched"
}
```

## 2. State Granularity

```rust
// Option A: monolithic state (simple, default for small apps)
#[derive(Clone)]
struct AppState {
    db: DbPool,
    config: AppConfig,
}

// Option B: extracted service layers (better for testing)
#[derive(Clone)]
struct AppState {
    users: UserService,
    posts: PostService,
    analytics: AnalyticsClient,
}

// Option C: state per route group via nested routers
fn health_router() -> Router<()> {
    Router::new().route("/health", get(|| async { "ok" }))
}

fn api_router(state: ApiState) -> Router {
    Router::new()
        .route("/items", get(list_items))
        .with_state(state)
}
```

Using `Router<()>` for routes that don't need state means you can merge them without wrapping.

## 3. Using Tower-HTTP Middleware Layers

`tower-http` provides standard implementations for CORS, compression, tracing, and timeouts.

```rust
use axum::{routing::get, Router};
use tower_http::{
    cors::{Any, CorsLayer},
    trace::TraceLayer,
    timeout::TimeoutLayer,
    compression::CompressionLayer,
    request_id::RequestIdLayer,
};
use std::time::Duration;

let cors = CorsLayer::new()
    .allow_origin(Any)
    .allow_methods(Any);

let app = Router::new()
    .route("/", get(|| async { "Hello" }))
    .layer(TraceLayer::new_for_http())
    .layer(CompressionLayer::new())
    .layer(RequestIdLayer::new())
    .layer(cors)
    .layer(TimeoutLayer::new(Duration::from_secs(30)));
```

### Layer Ordering

Layers wrap from bottom to top. The last `.layer()` call is the outermost wrapper:

```
Request → TimeoutLayer → CorsLayer → CompressionLayer → TraceLayer → Router → Response
```

TraceLayer should be near the top (outermost) to capture full request/response timing including rejections.

## 4. Writing Custom Middleware

```rust
use axum::{
    body::Body,
    http::{Request, Response},
    middleware::Next,
    response::IntoResponse,
};

async fn my_custom_middleware(
    req: Request<Body>,
    next: Next,
) -> impl IntoResponse {
    // Pre-request work
    println!("Request: {} {}", req.method(), req.uri().path());

    let response = next.run(req).await;

    // Post-request work
    response
}

// Static files middleware pattern
async fn serve_file_middleware(
    req: Request<Body>,
    next: Next,
) -> impl IntoResponse {
    let path = req.uri().path().to_owned();
    let response = next.run(req).await;

    if response.status() == StatusCode::NOT_FOUND && !path.starts_with("/api/") {
        // Serve index.html for SPA routing
        serve_index_html().await
    } else {
        response
    }
}
```

## 5. Middleware with State Access

```rust
use axum::{
    extract::{Request, State},
    middleware::Next,
    response::Response,
};

async fn auth_middleware(
    State(state): State<Arc<AppState>>,
    mut req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let token = req.headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let user = state.auth.validate_token(token).await.map_err(|_| StatusCode::UNAUTHORIZED)?;
    
    // Inject user into request extensions for downstream handlers
    req.extensions_mut().insert(user);
    
    Ok(next.run(req).await)
}
```

To use state-aware middleware, wrap it with `axum::middleware::from_fn_with_state`:

```rust
Router::new()
    .route("/protected", get(protected_handler))
    .route_layer(axum::middleware::from_fn_with_state(state.clone(), auth_middleware))
    .with_state(state);
```

## 6. Scoped Middleware

```rust
let app = Router::new()
    // No auth for public routes
    .route("/health", get(health))
    .route("/login", post(login))
    // Auth applied only to /api routes
    .nest("/api", api_routes().layer(auth_middleware));
```

Using `route_layer` vs `layer`:

- `route_layer`: applied only to routes defined *on that router* (not nested routers).
- `layer`: wraps the entire router including nested routers.
- `route_layer` + `nest`: middleware only applies to the nested group.

## 7. Fallible Middleware

```rust
use axum::response::IntoResponse;

#[derive(Debug)]
struct RateLimited;

impl IntoResponse for RateLimited {
    fn into_response(self) -> Response {
        (StatusCode::TOO_MANY_REQUESTS, "rate limit exceeded").into_response()
    }
}

async fn rate_limit_middleware(
    req: Request,
    next: Next,
) -> Result<Response, RateLimited> {
    if is_rate_limited(&req).await {
        return Err(RateLimited);
    }
    Ok(next.run(req).await)
}
```

When middleware returns `Err`, the rejection bypasses all inner layers — use this for short-circuit semantics.

## 8. Sharing State Between Middleware and Handlers

```rust
// Using request extensions
async fn inject_request_id(
    mut req: Request,
    next: Next,
) -> Response {
    let request_id = Uuid::new_v4();
    req.extensions_mut().insert(RequestId(request_id));
    let response = next.run(req).await;
    response
}

// Handler retrieves it
async fn handler(Extension(request_id): Extension<RequestId>) -> impl IntoResponse {
    format!("request id: {}", request_id.0)
}
```

## 9. Graceful Shutdown

```rust
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("install SIGTERM handler")
            .recv().await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}

// Usage
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await
    .unwrap();
```

## 10. Testing Middleware

```rust
use axum::{
    body::Body,
    http::{Request, StatusCode},
};
use tower::ServiceExt; // for .oneshot()

#[tokio::test]
async fn test_auth_middleware_rejects_missing_token() {
    let app = protected_routes();
    
    let response = app
        .oneshot(Request::builder()
            .uri("/api/data")
            .body(Body::empty())
            .unwrap())
        .await
        .unwrap();

    assert_eq!(response.status(), StatusCode::UNAUTHORIZED);
}

#[tokio::test]
async fn test_auth_middleware_accepts_valid_token() {
    let app = protected_routes();
    
    let response = app
        .oneshot(Request::builder()
            .uri("/api/data")
            .header("Authorization", "Bearer valid-token")
            .body(Body::empty())
            .unwrap())
        .await
        .unwrap();

    assert!(response.status().is_success());
}
```

## 11. Common Middleware Configurations

| Concern | Crate | Layer |
|---------|-------|-------|
| Logging | `tower-http` | `TraceLayer` |
| CORS | `tower-http` | `CorsLayer` |
| Compression | `tower-http` | `CompressionLayer` |
| Timeout | `tower-http` | `TimeoutLayer` |
| Rate limit | `tower-governor` / custom | Custom middleware |
| Request ID | `tower-http` | `RequestIdLayer` |
| Auth | Custom | Custom `from_fn_with_state` |
| Sentry | `sentry-tower` | `SentryLayer` |
| Prometheus | `axum-prometheus` | `PrometheusMetricLayer` |
| CSRF | `tower-http` | `CsrfLayer` |
