# Tracing in Rust

`tracing` is the standard path for production observability in Rust. It provides structured, typed, async-safe spans and events.

## Core Architecture

```text
events (log points) ─┐
spans (scopes) ──────┤── subscriber ── output (JSON, text, OTLP)
                      │
                      └── per-crate EnvFilter
```

Subscribers receive all `tracing` macro invocations. Multiple subscribers cannot coexist without a bridge layer.

## Subscriber Selection

| Crate | Use case |
|-------|----------|
| `tracing-subscriber` | stdout JSON/text/compact formatting + env-filter |
| `opentelemetry-otlp` | Send to OpenTelemetry collector |
| `tonic` | gRPC transport for OTLP |
| `tracing-appender` | Non-blocking file output |
| `tracing-subscriber::fmt` with `TestWriter` | Capture for test assertions |

## Spans

```rust
use tracing::{info_span, Span};

let parent = info_span!("request", request_id = %req_id, path = path);

// Inner span inherits parent context
let db_span = info_span!(parent: &parent, "db.query", table = %table_name);
```

Spans are scoped via `span.enter()` guard, `.instrument(future)`, or the `#[instrument]` attribute.

## Instrument Attribute

```rust
#[tracing::instrument(
    skip(password_hash, pool),
    fields(user_id = %user.id, tenant_id),
    err(level = Level::WARN),
)]
pub async fn login(&self, login: &Login, password_hash: &str) -> Result<User, AppError> {
    // skip: large or sensitive args not logged
    // fields: added to span, tenant_id set inside fn
    // err(level = WARN): log errors at WARN, successes at INFO (default)
}
```

Use `skip` for credentials, DB pools, configs, and large context bundles. Use `fields` to add structured data after the function starts.

## EnvFilter

```
RUST_LOG=info,my_crate=debug,tower_http=warn
my_crate=debug
```

EnvFilter supports direct crate levels, span field filters, and complex directive chains. Use `--` to separate directives for trace-logging libraries.

## Fields vs Messages

```rust
// Emit structured fields — queryable, filterable, indexable.
tracing::info!(user_id, duration_ms, "session ended");

// Avoid interpolating structured data into the message—hard to parse.
tracing::info!("user {} ended session in {}ms", user_id, duration_ms);
```

Structured fields combine message + key=value pairs into the subscriber's output format.

## Async Spans

Spans across `.await` points persist correctly because `tracing` uses an `Executor` that re-enters the span after poll. Always instrument futures spawned into a `JoinSet` or `tokio::spawn` to propagate context.

## Trace ID Propagation

```rust
use opentelemetry::global;
use tracing_opentelemetry::OpenTelemetrySpanExt;

let context = current_span.context();
let trace_id = context.span().span_context().trace_id();
```

Propagate the trace ID outbound in HTTP headers (`traceparent`, `tracestate`) and inbound via middleware extraction.

## Testing

```rust
use tracing_subscriber::fmt::TestWriter;

let subscriber = tracing_subscriber::fmt()
    .with_writer(TestWriter::new())
    .with_env_filter("test=debug")
    .finish();
tracing::subscriber::set_global_default(subscriber)?;
```

Keep one subscriber per test process. Use `tracing-test` or `tracing-subscriber::reload` for per-test filtering.

## Anti-Patterns

- Adding large payloads or binary data as span fields.
- Depending on interpolated `{}` for observability—loses structure.
- Unwrapping inside instrumented code that logs the panic—log before failure.
- Mixing `log` crate and `tracing` without the `tracing-log` bridge.
- Spawning tasks without instrumenting them—loses parent spans.
