# Axum Routing & Extractors: High-Performance Request Processing

Axum uses a declarative, type-safe routing model built on top of the `matchit` router. Handlers are regular async functions that receive input via types that implement `FromRequest` or `FromRequestParts`.

## 1. Type-Safe Routing & Route Matching

Routes in Axum are matched deterministically. The matching system does not depend on insertion order, but rather on path specificity.

```rust
use axum::{routing::get, Router};

let app = Router::new()
    .route("/users", get(list_users))
    .route("/users/new", get(new_user_form))
    .route("/users/:id", get(get_user));
```

### Route Specificity Rules:
- **Static segments** (e.g., `/users/new`) take precedence over **dynamic parameters** (e.g., `/users/:id`).
- Wildcards (`/*path`) match everything remaining and have the lowest priority.

## 2. Axum Extractors

Extractors allow handlers to request data from the incoming HTTP request. They are defined as function parameters and are evaluated from left to right.

### The Extractor Order Rule

> Axum handlers can have at most one extractor that consumes the request body (e.g., `Json`, `Form`, `Bytes`, `String`). This body extractor MUST be the last parameter in the handler signature.

```rust
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    Json,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;

#[derive(Deserialize)]
pub struct QueryParams {
    pub limit: Option<usize>,
}

#[derive(Deserialize)]
pub struct CreateUser {
    pub username: String,
    pub email: String,
}

#[derive(Serialize)]
pub struct UserResponse {
    pub id: u64,
    pub username: String,
}

pub struct AppState {
    pub db: DatabaseConnection,
}

// Correct: Query and Path extract parts from headers/URI; Json consumes the body and is last.
async fn create_user_handler(
    State(state): State<Arc<AppState>>,
    Path(group_id): Path<String>,
    Query(params): Query<QueryParams>,
    Json(payload): Json<CreateUser>,
) -> Result<(StatusCode, Json<UserResponse>), StatusCode> {
    let limit = params.limit.unwrap_or(10);
    let user = state.db.insert_user(&payload.username, &payload.email, &group_id)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok((StatusCode::CREATED, Json(UserResponse { id: user.id, username: user.username })))
}
```

## 3. Writing Custom Extractors

Implement `FromRequest` (for body consumers) or `FromRequestParts` (for header/metadata consumers).

```rust
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

pub struct Claims {
    pub user_id: String,
}

#[async_trait]
impl<S> FromRequestParts<S> for Claims
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let auth_header = parts.headers
            .get("Authorization")
            .and_then(|value| value.to_str().ok())
            .ok_or((StatusCode::UNAUTHORIZED, "missing authorization header"))?;

        if auth_header.starts_with("Bearer ") {
            let token = &auth_header[7..];
            Ok(Claims { user_id: "user_123".to_string() })
        } else {
            Err((StatusCode::UNAUTHORIZED, "invalid authorization scheme"))
        }
    }
}
```

## 4. Composing Multiple Extractors

Extractors that consume `RequestParts` (not body) can be combined freely. The order only matters for body extractors.

```rust
async fn handler(
    // Parts extractors — any order, any number
    Query(params): Query<Params>,
    Path(id): Path<Uuid>,
    headers: HeaderMap,
    cookies: CookieJar,
    State(state): State<AppState>,
    // Body extractor — must be last, at most one
    Json(payload): Json<Payload>,
) -> impl IntoResponse {
    // ...
}
```

## 5. Path Parameter Extraction Strategies

```rust
// Simple typed path
async fn get_user(Path(id): Path<Uuid>) -> impl IntoResponse { /* ... */ }

// Multiple path params as tuple
async fn get_post(Path((blog_id, post_id)): Path<(String, Uuid)>) -> impl IntoResponse { /* ... */ }

// Named struct — cleanest for 3+ params
#[derive(Deserialize)]
struct PostPath {
    blog_id: String,
    post_id: Uuid,
    revision: Option<i32>,
}
async fn get_post_version(Path(path): Path<PostPath>) -> impl IntoResponse { /* ... */ }
```

Axum deserializes path params using `serde`. Use `Option<T>` for optional path segments.

## 6. Query Parameter Patterns

```rust
// Required params — missing fields return 422
async fn search(Query(params): Query<SearchParams>) -> impl IntoResponse { /* ... */ }

// Optional params with defaults
#[derive(Deserialize)]
struct ListParams {
    #[serde(default = "default_page")]
    page: usize,
    #[serde(default)]
    per_page: Option<usize>,
    #[serde(default)]
    sort: Option<String>,
}

// Raw query string for passthrough
use axum::extract::RawQuery;
async fn proxy(RawQuery(query): RawQuery) -> impl IntoResponse { /* ... */ }
```

## 7. Nested Routing and Merging

```rust
use axum::{routing::{get, post}, Router};

fn admin_routes() -> Router<AppState> {
    Router::new()
        .route("/admin/users", get(list_users))
        .route("/admin/users/:id", delete(delete_user))
}

fn api_routes() -> Router<AppState> {
    Router::new()
        .route("/api/v1/items", get(list_items))
        .route("/api/v1/items", post(create_item))
}

// Merge routers — they share the same state type
let app = Router::new()
    .merge(admin_routes())
    .merge(api_routes());
```

For prefix grouping without merge:

```rust
let app = Router::new()
    .nest("/api/v1", api_routes())
    .nest("/admin", admin_routes());
```

`nest` strips the prefix before matching child routes. This matters for middleware scope and path extraction.

## 8. Fallback Routes

```rust
let app = Router::new()
    .route("/health", get(health_check))
    .fallback(handler_404);

async fn handler_404() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, Json(serde_json::json!({"error": "not found"})))
}
```

Per-router fallbacks compose: a nested router's fallback only applies within its scope.

## 9. Error Handling Patterns

```rust
// Rejection-based (cleanest for most services)
async fn handler() -> Result<Json<Data>, AppError> {
    let data = fetch_data().await.map_err(AppError::from)?;
    Ok(Json(data))
}

// IntoResponse for AppError
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let status = match &self {
            AppError::NotFound => StatusCode::NOT_FOUND,
            AppError::DbError(_) => StatusCode::INTERNAL_SERVER_ERROR,
            AppError::Validation(_) => StatusCode::BAD_REQUEST,
        };
        (status, Json(ErrorResponse { message: self.to_string() })).into_response()
    }
}
```

## 10. Performance Considerations

- Extractors run sequentially, left to right. Put the cheapest extractors (Path, Query) first, body extractors last.
- `State` extraction is zero-cost — it is a reference lookup, not a clone.
- `Extension<T>` uses type-map lookup — prefer `State` for app-scoped data.
- Large body extractors allocate before the handler runs. Consider streaming for large payloads.
- Use `axum::body::Body` directly for proxy/forwarding handlers that don't need to parse bodies.

## 11. Debugging Route Conflicts

```rust
// Axum will panic at startup on conflicting routes:
// "Invalid route: route `/users/:id` conflicts with `/users/{id}`"

// Fix: ensure dynamic params have unique names
.route("/users/:id", get(by_id))       // OK
.route("/users/:name", get(by_name));  // Conflict! Same pattern, different name
```

Use `axum::Router::new().route(...)` and test route resolution with integration tests.
