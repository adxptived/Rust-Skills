---
name: rust-cloud-native
description: |
  Rust Cloud-Native architectures and patterns. Use when:
  - Designing distributed services, microservices, or cloud applications
  - Integrating gRPC communication via Tonic
  - Implementing observability (structured logging, tracing, metrics)
  - Setting up Docker multi-stage builds or containerizing Rust binaries
  - Setting up graceful shutdown behaviors or health probes
  - Configuring Kubernetes deployments, probes, HPA, or service meshes
  - Implementing service discovery, config reload, or feature flags
  - Designing CI/CD pipelines for Rust cloud services
---

## Quick Navigation

- [references/observability.md](references/observability.md)
- [references/deployments.md](references/deployments.md)

# Rust Cloud-Native Architecture Guide

Best practices for building containerized, observable, and highly reliable microservices in Rust.

## Core Rules & Patterns

### 1. Structured Logging & Tracing
Use the `tracing` ecosystem instead of `log` or standard prints. Incorporate OpenTelemetry integration for distributed request context propagation.

```rust
use tracing::{info, instrument};

#[instrument(skip(db), fields(user_id = %payload.user_id))]
pub async fn process_order(payload: OrderPayload, db: &DbConnection) -> Result<(), AppError> {
    info!("Processing new incoming order");
    db.save_order(&payload).await?;
    info!("Order saved successfully");
    Ok(())
}
```

### 2. High-Performance APIs: gRPC with Tonic
Use the `tonic` crate to define strict APIs via Protocol Buffers.

```rust
// proto definition (build.rs handles codegen)
// service OrderService { rpc CreateOrder (OrderRequest) returns (OrderResponse); }

use tonic::{Request, Response, Status};
use pb::order_service_server::OrderService;

pub struct MyOrderService;

#[tonic::async_trait]
impl OrderService for MyOrderService {
    async fn create_order(
        &self,
        request: Request<OrderRequest>,
    ) -> Result<Response<OrderResponse>, Status> {
        let req = request.into_inner();
        // business logic ...
        Ok(Response::new(OrderResponse { success: true }))
    }
}
```

### 3. Graceful Shutdown
Ensure the server handles orchestration signals (`SIGINT`, `SIGTERM`) gracefully to finish flight requests before exiting.

```rust
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to listen for ctrl+c");
    };

    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM signal handler")
            .recv()
            .await;
    };

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
    println!("Graceful shutdown signal received. Stopping worker pools...");
}
```

### 4. Health Probes
Expose lightweight health checkpoints for liveness and readiness states.

```rust
use axum::{routing::get, Router, http::StatusCode};

async fn liveness() -> StatusCode {
    StatusCode::OK
}

async fn readiness() -> Result<StatusCode, StatusCode> {
    // Check database connection or dependencies here
    if db_pool_is_ok().await {
        Ok(StatusCode::OK)
    } else {
        Err(StatusCode::SERVICE_UNAVAILABLE)
    }
}
```

### 5. Multi-Stage Distroless Docker Builds

```dockerfile
# Build Stage
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# Runner Stage
FROM gcr.io/distroless/cc-debian12
COPY --from=builder /app/target/release/my-service /my-service
USER 10001:10001
ENTRYPOINT ["/my-service"]
```

## Configuration Management

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize, Clone)]
pub struct Config {
    pub database_url: String,
    pub redis_url: String,
    pub port: u16,
    pub log_level: String,
    pub feature_flags: FeatureFlags,
}

#[derive(Debug, Deserialize, Clone)]
pub struct FeatureFlags {
    pub enable_new_pipeline: bool,
    pub max_batch_size: usize,
}

impl Config {
    pub fn from_env() -> Result<Self, config::ConfigError> {
        config::Config::builder()
            .add_source(config::Environment::default().prefix("APP"))
            .build()?
            .try_deserialize()
    }
}
```

Use environment variables or a config service. Avoid config files in container images — they break twelve-factor principles.

## Service Architecture Decisions

### REST vs gRPC

Use REST/JSON for public APIs, browser clients, and human-debuggable integrations. Use gRPC for internal service-to-service APIs where strict schemas, streaming, and generated clients matter.

```rust
// REST edge service: axum + serde + tower-http
// Internal RPC: tonic + prost + OpenTelemetry propagation
```

### Monolith vs Microservice

Start with a modular service unless independent scaling, ownership, or deployment cadence justifies a separate service. Network boundaries add retries, tracing, auth, schema evolution, and failure modes.

## Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-rust-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-rust-service
  template:
    metadata:
      labels:
        app: my-rust-service
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        image: my-registry/my-rust-service:latest
        ports:
        - containerPort: 3000
        - containerPort: 9001  # metrics
        env:
        - name: APP_PORT
          value: "3000"
        - name: APP_DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 3
          periodSeconds: 5
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

## Runtime Rules

- Use Tokio for network services.
- Bound all queues and channels.
- Add timeouts around outbound calls.
- Propagate cancellation on shutdown.
- Emit structured logs with request IDs.
- Expose readiness and liveness endpoints.
- Keep secrets in environment/platform secret stores, not config files.
- Configure resource requests/limits in Kubernetes.
- Use `terminationGracePeriodSeconds` to allow in-flight requests to finish.

## Resilience Patterns

```rust
let response = tokio::time::timeout(
    Duration::from_secs(2),
    client.fetch_user(user_id),
).await??;
```

Add retries only for idempotent operations or operations protected by idempotency keys with exponential backoff.

### Circuit Breaker

```rust
use tokio::sync::Semaphore;

pub struct CircuitBreaker {
    semaphore: Semaphore,
    failures: AtomicUsize,
    threshold: usize,
}

impl CircuitBreaker {
    pub async fn call<T, F, E>(&self, f: F) -> Result<T, Error>
    where
        F: Future<Output = Result<T, E>>,
    {
        if self.failures.load(Ordering::Acquire) >= self.threshold {
            return Err(Error::CircuitOpen);
        }

        let _permit = self.semaphore.acquire().await?;
        match f.await {
            Ok(val) => {
                self.failures.store(0, Ordering::Release);
                Ok(val)
            }
            Err(err) => {
                self.failures.fetch_add(1, Ordering::Release);
                Err(err.into())
            }
        }
    }
}
```

## Feature Flags

```rust
pub enum FeatureFlag {
    NewBillingPipeline,
    OptimizedSearch,
}

pub struct FlagService {
    flags: Arc<RwLock<HashMap<FeatureFlag, bool>>>,
}

impl FlagService {
    pub fn is_enabled(&self, flag: FeatureFlag) -> bool {
        self.flags.read().get(&flag).copied().unwrap_or(false)
    }

    /// Reload flags from config source without restart.
    pub async fn reload(&self) -> Result<(), Error> {
        let new_flags = load_flags_from_source().await?;
        *self.flags.write() = new_flags;
        Ok(())
    }
}
```

## Deployment Checklist

- `cargo build --release --locked` in CI.
- Multi-stage Docker image with distroless base.
- Non-root container user.
- Health probes configured (liveness + readiness).
- SIGTERM graceful shutdown tested.
- Metrics include request rate, errors, duration, saturation.
- Logs are JSON in production.
- Dashboards and alerts exist before launch.
- Resource requests/limits set in Kubernetes.
- PodDisruptionBudget configured for multi-replica services.
- Config and secrets separated from code.
- CI/CD pipeline includes `cargo audit` and `cargo test`.

## Anti-Patterns

- Unbounded `mpsc` channels in request paths.
- `println!` logging in services.
- Retry loops without jitter/backoff.
- Readiness endpoint that always returns 200.
- Database migrations hidden in request startup without observability.
- One global timeout for all downstream dependencies.
- Hardcoded service addresses (use DNS/service discovery).
- Config files baked into container images.

## Review Prompt

When reviewing cloud-native Rust, check shutdown behavior, timeouts, bounded resources, observability coverage, config/secrets handling, and container reproducibility.

## References

- [Tokio graceful shutdown topic](https://tokio.rs/tokio/topics/shutdown)
- [OpenTelemetry Rust](https://opentelemetry.io/docs/languages/rust/)
- [Kubernetes probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Twelve-factor app](https://12factor.net/)
- [Kubernetes Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
