# Observability for Rust Services

Production services need correlated logs, metrics, traces, and health signals. The three pillars — logs, metrics, traces — must be correlated by request ID or trace ID.

## Structured Logs with Tracing

Use `tracing`, not `println!` or `log`.

```rust
use tracing::{info, warn, error, instrument};

#[instrument(skip(state), fields(order_id = %order.id))]
pub async fn submit_order(state: AppState, order: Order) -> Result<(), Error> {
    info!(customer_id = %order.customer_id, "submitting order");
    state.store.save(order).await?;
    info!("order saved");
    Ok(())
}
```

### Span Fields to Always Include

| Field | Example | Why |
|-------|---------|-----|
| `request_id` | `req-abc123` | Trace across services |
| `service` | `orders-api` | Identify source |
| `environment` | `production` | Filter by env |
| `customer_id` | `cust_456` | Business context |
| `duration_ms` | `42` | Latency per operation |

## Subscriber Setup

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing() {
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer().json())
        .init();
}
```

### Layered Subscriber for Dev vs Prod

```rust
pub fn init_tracing(environment: &str) {
    let env_filter = tracing_subscriber::EnvFilter::from_default_env();

    // JSON formatter for production, pretty for dev
    let fmt_layer = if environment == "production" {
        tracing_subscriber::fmt::layer()
            .json()
            .with_target(true)
            .with_current_span(true)
            .boxed()
    } else {
        tracing_subscriber::fmt::layer()
            .pretty()
            .with_target(true)
            .with_file(true)
            .with_line_number(true)
            .boxed()
    };

    tracing_subscriber::registry()
        .with(env_filter)
        .with(fmt_layer)
        .init();
}
```

Set `RUST_LOG=my_service=info,tower_http=debug` in deployments.

## Dynamic Log Level Control

```rust
use tracing_subscriber::reload;

pub fn init_dynamic_logging() -> reload::Handle<EnvFilter, tracing_subscriber::Registry> {
    let filter = EnvFilter::from_default_env();
    let (filter_layer, reload_handle) = reload::Layer::new(filter);

    tracing_subscriber::registry()
        .with(filter_layer)
        .with(tracing_subscriber::fmt::layer().json())
        .init();

    reload_handle
}

// At runtime, change log level via HTTP or signal:
fn handle_log_level_change(handle: &reload::Handle<EnvFilter, tracing_subscriber::Registry>, level: &str) {
    let new_filter = EnvFilter::new(level);
    handle.reload(new_filter).expect("reload failed");
}
```

Useful for production debugging: POST to `/admin/log-level?level=debug` to increase verbosity temporarily.

## Request IDs

Attach request IDs at ingress and propagate them through spans.

```rust
use tower_http::request_id::{MakeRequestId, RequestId, RequestIdLayer};
use uuid::Uuid;

#[derive(Clone)]
struct UuidRequestIdMaker;

impl MakeRequestId for UuidRequestIdMaker {
    fn make_request_id<B>(&mut self, _request: &axum::http::Request<B>) -> Option<RequestId> {
        let id = Uuid::new_v4().to_string();
        Some(RequestId::new(id.parse().unwrap()))
    }
}

let app = Router::new()
    .layer(RequestIdLayer::new(
        MakeSetRequestIdLayer::new(
            HeaderName::from_static("x-request-id"),
            UuidRequestIdMaker,
        ),
    ));
```

## Metrics

Expose Prometheus metrics for latency, error rate, queue depth, and saturation.

```rust
metrics::counter!("http_requests_total", "route" => "/orders").increment(1);
metrics::histogram!("http_request_duration_seconds").record(elapsed.as_secs_f64());
```

### Axum Metrics Middleware

```rust
use axum_prometheus::PrometheusMetricLayer;

let (prometheus_layer, metric_handle) = PrometheusMetricLayer::pair();

let app = Router::new()
    .route("/metrics", get(|| async move { metric_handle.render() }))
    .layer(prometheus_layer);
```

### Custom Metrics

```rust
use metrics::{describe_counter, describe_histogram, counter, histogram};
use once_cell::sync::Lazy;

static ORDERS_CREATED: Lazy<Counter> = Lazy::new(|| {
    let c = counter!("orders_created_total");
    describe_counter!("orders_created_total", "Total orders created");
    c
});

static ORDER_DURATION: Lazy<Histogram> = Lazy::new(|| {
    let h = histogram!("order_duration_seconds");
    describe_histogram!("order_duration_seconds", "Order processing time");
    h
});

async fn create_order() {
    let start = std::time::Instant::now();
    // ... process order ...
    ORDERS_CREATED.increment(1);
    ORDER_DURATION.record(start.elapsed().as_secs_f64());
}
```

### Recommended Metrics per Service

| Metric name | Type | Labels |
|------------|------|--------|
| `http_requests_total` | Counter | `route`, `method`, `status` |
| `http_request_duration_seconds` | Histogram | `route` |
| `db_queries_total` | Counter | `statement`, `status` |
| `db_query_duration_seconds` | Histogram | `statement` |
| `connections_active` | Gauge | `pool` |
| `queue_depth` | Gauge | `queue` |
| `errors_total` | Counter | `error_kind` |
| `cache_hits_total` | Counter | `cache` |
| `cache_misses_total` | Counter | `cache` |

### Label Cardinality Warning

Keep label cardinality bounded. Never use user IDs, request IDs, or random values as metric labels — every unique label combination creates a new time series.

```rust
// Bad: unbounded cardinality (each user_id creates a new series)
counter!("api_calls", "user_id" => user_id).increment(1);

// Good: bounded labels
counter!("api_calls", "tier" => user_tier).increment(1);
```

## Traces

Use OpenTelemetry when requests cross service boundaries. Propagate `traceparent` headers and export to an OTLP collector.

```rust
use opentelemetry::global;
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::trace::Sampler;

fn init_opentelemetry() {
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_trace_config(
            opentelemetry_sdk::trace::Config::default()
                .with_sampler(Sampler::TraceIdRatioBased(0.1))
        )
        .with_exporter(opentelemetry_otlp::new_exporter().tonic()
            .with_endpoint("http://otel-collector:4317"))
        .install_batch(opentelemetry_sdk::runtime::Tokio)
        .expect("opentelemetry init");

    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(tracing_opentelemetry::layer().with_tracer(tracer))
        .init();
}
```

### Trace Sampling Strategy

| Strategy | When to use | Sampling rate |
|----------|------------|---------------|
| Head-based | Simple, stateless | 1-10% of requests |
| Tail-based | Error/latency focus | 100% sampled, then filter |
| Rate-limited | Budget-constrained | Fixed traces/sec |
| Dynamic | Cost-sensitive | Vary by route/endpoint |

### Propagating Context

```rust
use opentelemetry::global::get_text_map_propagator;
use opentelemetry::propagation::TextMapPropagator;

// Inject into outbound request headers
let mut headers = reqwest::header::HeaderMap::new();
let propagator = get_text_map_propagator(|p| {
    p.inject(&mut HeaderMapInjector(&mut headers))
});
let response = client.get(url).headers(headers).send().await?;

// Extract from inbound request
let parent_context = get_text_map_propagator(|p| {
    p.extract(&HeaderMapExtractor(&incoming_headers))
});
let span = tracer.start_with_context("process", &parent_context);
```

## Health Signals

```rust
use axum::{routing::get, Json, Router};
use std::sync::atomic::{AtomicBool, Ordering};

struct HealthState {
    ready: AtomicBool,
    started: AtomicBool,
}

async fn healthz() -> &'static str { "ok" }

async fn readyz(State(state): State<Arc<HealthState>>) -> Result<&'static str, StatusCode> {
    state.ready.load(Ordering::Acquire)
        .then(|| "ok")
        .ok_or(StatusCode::SERVICE_UNAVAILABLE)
}

async fn startupz(State(state): State<Arc<HealthState>>) -> Result<&'static str, StatusCode> {
    state.started.load(Ordering::Acquire)
        .then(|| "ok")
        .ok_or(StatusCode::SERVICE_UNAVAILABLE)
}
```

- **Liveness**: process is alive (cheap, no dependencies)
- **Readiness**: process can serve traffic (checks dependencies)
- **Startup**: process finished migrations/cache warmup

Keep probes cheap and bounded with timeouts.

## Correlation: Connecting the Three Pillars

```rust
// Each log/span/metric carries the same trace_id
pub struct CorrelationFields {
    pub trace_id: String,
    pub service: &'static str,
    pub version: &'static str,
    pub environment: &'static str,
}

// In every log line
info!(
    trace_id = %correlation.trace_id,
    service = %correlation.service,
    version = %correlation.version,
    environment = %correlation.environment,
    "order processed"
);
```

Without correlation fields, logs, metrics, and traces exist in separate silos. A single dashboard query should connect them via `trace_id` or `request_id`.

## Alert Rules (Prometheus)

```yaml
groups:
  - name: rust-service
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 5% for {{ $labels.service }}"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 3m
        labels:
          severity: warning

      - alert: NoTraffic
        expr: rate(http_requests_total[5m]) == 0
        for: 15m
        labels:
          severity: warning

      - alert: QueueDepth
        expr: queue_depth > 1000
        for: 2m
        labels:
          severity: critical
```

## Observability Anti-Patterns

- Logging at `ERROR` level for minor recoverable issues (noise → ignored alerts).
- Missing `trace_id` / `request_id` on log lines.
- Metric label cardinality explosion (user IDs, timestamps as labels).
- Health endpoint that never returns non-200.
- `panic!` without a panic hook to log the stack trace.

```rust
// Set up a panic hook that logs via tracing
use std::panic;

fn set_panic_hook() {
    panic::set_hook(Box::new(|panic_info| {
        tracing::error!(
            panic_info = %panic_info,
            "process panicked"
        );
    }));
}
```
