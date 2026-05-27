# Metrics in Rust

`metrics` is the standard facade for emitting metrics in Rust applications. It supports counters, gauges, histograms, and labels with `metrics-exporter-prometheus`, `metrics-exporter-otlp`, and other backends.

## Core Types

| Type | Use case |
|------|----------|
| `counter!` | Monotonically increasing value (requests, errors, bytes) |
| `gauge!` | Point-in-time value (queue depth, connections, memory) |
| `histogram!` | Distribution of values (latency, payload size, batch count) |

## RED / USE Methodology

| Pattern | Metric | Type |
|---------|--------|------|
| **R**ate | Requests per second | `counter!` |
| **E**rrors | Failed requests per second | `counter!` with `status` label |
| **D**uration | Latency distribution | `histogram!` |
| **U**tilization | Resource busy time | `gauge!` |
| **S**aturation | Queued/waiting work | `gauge!` |
| **E**rrors | Resource operation failures | `counter!` |

## Basic Usage

```rust
use metrics::{counter, gauge, histogram};

counter!("http_requests_total", "method" => "GET", "status" => "200").increment(1);
gauge!("connections_active").set(current as f64);
histogram!("request_duration_seconds", "route" => "/api/users").record(0.042);
```

## Prometheus Exporter

```toml
[dependencies]
metrics = "0.23"
metrics-exporter-prometheus = "0.16"
```

```rust
use metrics_exporter_prometheus::{PrometheusBuilder, PrometheusHandle};

pub fn setup_metrics() -> PrometheusHandle {
    PrometheusBuilder::new()
        .listen_address("0.0.0.0:9001".parse().unwrap())
        .install()
        .expect("prometheus exporter")
}
```

For private metric endpoints, bind to localhost or embed the handle in the app and serve via Axum.

## Axum Metrics Middleware

```rust
use metrics::{counter, histogram};
use std::time::Instant;

pub async fn metrics_middleware<B>(
    req: axum::http::Request<B>,
    next: axum::middleware::Next<B>,
) -> axum::response::Response {
    let start = Instant::now();
    let method = req.method().to_string();
    let uri = req.uri().path().to_owned();

    let response = next.run(req).await;

    let status = response.status().as_u16();
    let duration = start.elapsed().as_secs_f64();

    counter!("http_requests_total", "method" => &method, "status" => status.to_string()).increment(1);
    histogram!("http_request_duration_seconds", "method" => &method, "route" => &uri).record(duration);

    response
}
```

Always record duration after the response is produced, not before. Use route templates instead of raw paths to avoid unbounded label cardinality.

## Label Cardinality

Labels are the most common source of metrics misconfiguration.

```rust
// Bad: unbounded — user_id creates a new series per user.
counter!("api_calls", "user_id" => user.id.to_string()).increment(1);

// Good: bounded — route template has a fixed set of values.
counter!("api_calls", "route" => "/users/:id").increment(1);

// Acceptable: error code is naturally bounded.
counter!("db_errors", "code" => db_error.code()).increment(1);
```

Bounded labels: method, route template, status code class, error category, service name, version, deployment zone.

Unbounded labels: user ID, session ID, request ID, email, IP address, raw URL path, timestamp, random value.

## Histogram Configuration

```rust
use metrics_util::MetricKindMask;
use metrics_util::layers::Histogram;

PrometheusBuilder::new()
    .set_buckets_for_metric(
        MetricKindMask::HISTOGRAM,
        |_| Some(vec![
            0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0,
        ]),
    )
    .install()
    .expect("histogram with custom buckets");
```

Match buckets to expected latency range. Spend buckets on the range where most requests fall, not on outlier extremes.

## Testing Metrics

```rust
use metrics::{counter, increment_counter};
use metrics_util::debugging::DebugValue;

fn record_metric() {
    counter!("test_calls", "status" => "ok").increment(1);
}

#[test]
fn test_metric_recording() {
    let recorder = metrics_util::debugging::DebuggingRecorder::new();
    let snapshotter = recorder.snapshotter();
    metrics::set_recorder(recorder).unwrap();

    record_metric();

    let snapshot = snapshotter.snapshot();
    // Assert on counter values from the snapshot
}
```

In integration tests, use a separate recorder per test scope or a global recorder with `metrics-util` layering to avoid collision.

## Multi-Process Metrics

For multi-process deployments (sidecars, workers), consider:

- Pushgateway for short-lived batch jobs.
- `process_collector` for per-process metrics with instance label.
- OTLP exporter to send to a collector that deduplicates.

## What to Measure

**Service level:**
- Request rate, error rate, latency (p50, p95, p99) per route
- Active requests, connection count
- Queue depth, batch sizes, processing time per item

**Dependency level:**
- Database query latency, connection pool utilization
- External API call latency and error rate
- Cache hit/miss ratio

**System level:**
- CPU, memory, file descriptors, goroutine/task count
- GC pause duration, allocated bytes per operation

## Anti-Patterns

```rust
// Bad: raw path as label — explodes cardinality.
histogram!("latency", "path" => req.uri().path().to_owned()).record(elapsed);

// Good: route template — bounded values.
histogram!("latency", "route" => "/users/:id").record(elapsed);
```

```rust
// Bad: recording before the operation — skews duration.
histogram!("query_time").record(0.0);
let _ = query_db().await;
histogram!("query_time").record(elapsed);

// Good: measure around the operation.
let start = Instant::now();
let result = query_db().await;
histogram!("query_time").record(start.elapsed());
```

## Metrics Checklist

- Route templates used as labels, not raw paths.
- Histogram buckets match expected latency range.
- Label cardinality is bounded and documented.
- Metrics endpoint is authenticated or bound to localhost in production.
- Every external dependency has latency, error rate, and throughput.
- Multi-process architecture has a plan for metric collection.
- Tests verify metric recording for critical paths.

## References

- [metrics crate](https://docs.rs/metrics)
- [metrics-exporter-prometheus](https://docs.rs/metrics-exporter-prometheus)
- [Prometheus best practices](https://prometheus.io/docs/practices/naming/)
- [RED method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
- [USE method](https://www.brendangregg.com/usemethod.html)
