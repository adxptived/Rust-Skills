---
name: rust-observability
description: |
  Rust observability with tracing, metrics, logs, OpenTelemetry, and production diagnostics. Use when:
  - User adds logging, tracing spans, metrics, request IDs, or distributed tracing to Rust services
  - User uses tracing, tracing-subscriber, tracing-opentelemetry, opentelemetry, metrics, prometheus, or axum tracing layers
  - User debugs missing logs, noisy logs, span propagation, trace IDs, structured fields, or async task visibility
  - User needs RED/USE metrics, histograms, dashboards, alerts, or SLO instrumentation
  - User instruments Axum/Tokio/background workers/databases/queues
  - User asks what to log, what not to log, or how to avoid leaking secrets/PII in telemetry
  - User sets up OTLP export to Jaeger, Grafana Tempo, Datadog, or Honeycomb
---

# Rust Observability

Practical guide for making Rust systems diagnosable in production: structured logs, spans, metrics, traces, and safe telemetry boundaries.

## Quick Navigation

- **references/tracing.md** - tracing setup, spans, subscribers, context propagation
- **references/metrics.md** - RED/USE metrics, histograms, labels, Prometheus/OpenTelemetry

## Golden Rules

1. Emit structured fields, not interpolated strings.
2. Put correlation IDs and domain IDs on spans once, then inherit them.
3. Measure latency with histograms, not averages.
4. Keep label cardinality bounded.
5. Never log secrets, credentials, bearer tokens, or raw sensitive payloads.

## Tracing Setup

```rust
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info,tower_http=info"));

    tracing_subscriber::fmt()
        .with_env_filter(filter)
        .json()
        .with_current_span(true)
        .with_span_list(true)
        .init();

    Ok(())
}
```

Use JSON logs in production and pretty logs locally. Configure log level through `RUST_LOG` or service config.

## Layered Subscriber Pattern

Build a subscriber from reusable layers for production setups:

```rust
use tracing_subscriber::{fmt, registry, EnvFilter, layer::SubscriberExt};

pub fn init_observability() -> Result<(), Box<dyn std::error::Error>> {
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));

    let subscriber = registry()
        .with(filter)
        .with(fmt::Layer::default().json());

    // Add OpenTelemetry layer when configured
    // .with(tracing_opentelemetry::layer().with_tracer(otlp_tracer));

    subscriber.try_init()?;
    Ok(())
}
```

This pattern lets you compose logging, OpenTelemetry, and custom layers independently.

## Dynamic Log Level Reloading

Change log levels at runtime without restart:

```rust
use tracing_subscriber::{fmt, reload, EnvFilter, Registry};

pub struct LogLevelHandle {
    handle: reload::Handle<EnvFilter, Registry>,
}

impl LogLevelHandle {
    pub fn set_level(&self, level: &str) -> Result<(), Box<dyn std::error::Error>> {
        let new_filter = EnvFilter::try_new(level)?;
        self.handle.reload(new_filter)?;
        Ok(())
    }
}

pub fn init_dynamic_logging() -> LogLevelHandle {
    let filter = EnvFilter::new("info");
    let (filter_layer, handle) = reload::Layer::new(filter);

    tracing_subscriber::registry()
        .with(filter_layer)
        .with(fmt::Layer::default().json())
        .init();

    LogLevelHandle { handle }
}
```

Expose this through an admin endpoint or signal handler for on-call debugging.

## Instrument Async Work

```rust
use tracing::{info_span, Instrument};

pub async fn handle_job(job: Job) -> Result<(), Error> {
    let span = info_span!(
        "job.process",
        job_id = %job.id,
        tenant_id = %job.tenant_id,
        kind = %job.kind,
    );

    async move {
        validate(&job)?;
        process(job).await?;
        Ok(())
    }
    .instrument(span)
    .await
}
```

Async tasks do not automatically inherit all context when spawned. Instrument spawned futures explicitly.

## Tracing-Aware Error Reporting

Capture span context in errors for richer diagnostics:

```rust
use tracing::Span;

#[derive(thiserror::Error, Debug)]
pub enum AppError {
    #[error("job {job_id} failed in tenant {tenant_id}: {source}")]
    JobFailed {
        job_id: String,
        tenant_id: String,
        source: Box<dyn std::error::Error + Send + Sync>,
    },
}

impl AppError {
    pub fn with_span_context(self, span: &Span) -> Self {
        // Copy span fields into the error for structured logging
        self
    }
}
```

## Axum Request Tracing

```rust
use tower_http::trace::{DefaultMakeSpan, DefaultOnResponse, TraceLayer};
use tracing::Level;

let app = Router::new()
    .route("/health", get(health))
    .layer(
        TraceLayer::new_for_http()
            .make_span_with(
                DefaultMakeSpan::new()
                    .level(Level::INFO)
                    .include_headers(false),
            )
            .on_response(
                DefaultOnResponse::new()
                    .level(Level::INFO)
                    .latency_unit(tower_http::LatencyUnit::Micros),
            ),
    );
```

Do not log request/response headers blindly; authorization and cookies are sensitive.

## OpenTelemetry OTLP Export

Send traces to a collector or observability backend:

```rust
use opentelemetry::{
    global,
    sdk::{propagation::TraceContextPropagator, trace as sdktrace},
};
use opentelemetry_otlp::WithExportConfig;

pub fn init_otlp(endpoint: &str) -> Result<(), Box<dyn std::error::Error>> {
    global::set_text_map_propagator(TraceContextPropagator::new());

    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_endpoint(endpoint),
        )
        .with_trace_config(
            sdktrace::config().with_resource(opentelemetry::sdk::Resource::new([
                opentelemetry::KeyValue::new("service.name", "my-service"),
            ])),
        )
        .install_batch(opentelemetry::runtime::Tokio)?;

    tracing_opentelemetry::layer().with_tracer(tracer);
    Ok(())
}
```

Use batch export for production; switch to simple export for local debugging.

## OTLP Sampling Strategies

| Strategy | Use Case | Effect |
|----------|----------|--------|
| AlwaysSample | Dev, critical paths | Every trace sent |
| TraceIdRatioBased | High-traffic production | Sample N% of all traces |
| ParentBased | Distributed tracing | Inherit sampling decision from parent span |
| RateLimiting | Cost control | Cap spans per second |

Configure sampling in the OTLP exporter pipeline:

```rust
use opentelemetry::sdk::trace::Sampler;

let config = sdktrace::config()
    .with_sampler(Sampler::TraceIdRatioBased(0.1)) // 10% sampling
    .with_resource(/* ... */);
```

## Metrics Setup

```rust
use metrics::counter;
use metrics_exporter_prometheus::PrometheusBuilder;

pub fn init_metrics() -> Result<(), Box<dyn std::error::Error>> {
    let builder = PrometheusBuilder::new();
    builder
        .with_http_listener("0.0.0.0:9001") // separate metrics port
        .install()?;
    Ok(())
}
```

Expose metrics on a separate port to avoid mixing with application traffic.

## Custom Metrics

```rust
use metrics::{counter, gauge, histogram, lazy_static};
use std::sync::LazyLock;

static REQUEST_COUNT: LazyLock<counter::Counter> = LazyLock::new(|| {
    counter!("http_requests_total", "method" => "GET", "route" => "/users/:id")
});

static REQUEST_DURATION: LazyLock<histogram::Histogram> = LazyLock::new(|| {
    histogram!("http_request_duration_seconds", "route" => "/users/:id")
});
```

Prefer lazily-initialized static metrics to avoid repeated label allocations on hot paths.

## Metrics Shape

```rust
// Counter: things that only go up
metrics::counter!("http_requests_total", "method" => "GET", "route" => "/users/:id").increment(1);

// Histogram: latency distributions
metrics::histogram!("http_request_duration_seconds", "route" => "/users/:id").record(elapsed.as_secs_f64());

// Gauge: current state
metrics::gauge!("worker_queue_depth", "queue" => "emails").set(depth as f64);
```

Use route templates, not raw paths. Raw paths turn IDs into unbounded label cardinality.

## Recommended Metrics

| Metric | Type | Labels | Purpose |
|--------|------|--------|---------|
| `http_requests_total` | Counter | method, route, status | Throughput |
| `http_request_duration_seconds` | Histogram | route, status | Latency (p50/p95/p99) |
| `http_requests_in_flight` | Gauge | route | Saturation |
| `db_query_duration_seconds` | Histogram | query_name | Dependency latency |
| `worker_queue_depth` | Gauge | queue_name | Backlog |
| `errors_total` | Counter | error_type, domain | Error rate |
| `pool_connections_used` | Gauge | pool_name | Resource usage |

## What to Log

- Request ID, trace ID, route template, status code, latency bucket.
- Domain IDs that are safe to expose internally.
- Retry count, timeout name, queue name, dependency name.
- Error category and stable error code.
- Key decision points with the inputs that drove the decision.

## What Not to Log

- Passwords, tokens, cookies, API keys, private keys.
- Full payment data, raw PII, auth headers, session contents.
- Large payloads that cause cost or privacy problems.
- High-cardinality dynamic values as metric labels.
- Stack traces for expected/business-logic errors (log at debug level instead).

## Structured Fields Convention

```rust
// Use stable field names across your codebase
tracing::info!(
    user_id = %uid,
    tenant_id = %tenant,
    order_id = %order,
    amount = amount_minor,
    currency = %currency,
    "order created"
);
```

Agree on field naming conventions per team. Inconsistent names break queries across services.

## Error Instrumentation

```rust
// Bad: error lost in a generic log
tracing::warn!("operation failed: {:?}", err);

// Good: capture typed error as a field
tracing::warn!(
    error_type = %err.kind(),
    error = %err,
    "operation failed"
);
```

When errors cross service boundaries, include a stable error code, not just a message string.

## Prometheus Alert Rules

```yaml
# alert-rules.yml
groups:
  - name: rust-service
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "Error rate {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "p95 latency {{ $value }}s"

      - alert: PoolExhaustion
        expr: pool_connections_used / pool_connections_max > 0.8
        for: 2m
        labels: { severity: warning }
```

## Anti-Patterns

```rust
// Bad: unstructured and hard to query.
println!("user {} failed: {:?}", user_id, err);

// Good: structured fields with stable keys.
tracing::warn!(%user_id, error = %err, "user operation failed");
```

```rust
// Bad: raw path label explodes cardinality.
metrics::counter!("requests", "path" => request.uri().path().to_owned()).increment(1);

// Good: route template has bounded values.
metrics::counter!("requests", "route" => "/users/:id").increment(1);
```

```rust
// Bad: blocking the async runtime with synchronous I/O
// inside `tracing::instrument` on a hot path.

// Good: keep span creation cheap; defer expensive formatting
// to subscriber layers (which run on their own thread).
```

```rust
// Bad: logging the same error at multiple levels
tracing::error!("operation failed");
// ... then re-thrown and caught again
tracing::warn!("operation also failed");

// Good: log at the boundary where recovery decision happens.
// Inner layers should propagate, not log.
```

## Production Checklist

- JSON logs include timestamp, level, target, span context, trace/request ID.
- Every external request path has latency, error, and throughput metrics.
- Metrics labels are bounded and documented.
- Spawned tasks are instrumented with parent/domain context.
- Secrets and PII are redacted at source, not only in downstream pipelines.
- Dashboards and alerts use symptoms first, internals second.
- OTLP exporter has sampling configured for production traffic volume.
- Log levels are dynamically changeable without redeploy.
- Error rates and latency have alert thresholds with runbooks.
- Every public endpoint has at least one RED metric.

## References

- [tracing crate](https://docs.rs/tracing)
- [tracing-subscriber](https://docs.rs/tracing-subscriber)
- [tower-http TraceLayer](https://docs.rs/tower-http/latest/tower_http/trace/)
- [OpenTelemetry Rust](https://opentelemetry.io/docs/languages/rust/)
- [Prometheus best practices](https://prometheus.io/docs/practices/naming/)
- [opentelemetry-otlp](https://docs.rs/opentelemetry-otlp)
