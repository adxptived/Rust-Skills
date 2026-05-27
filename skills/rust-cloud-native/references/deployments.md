# Deploying Rust Services

Cloud-native Rust deployments should be reproducible, small, observable, and graceful.

## Multi-Stage Dockerfile

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release --locked

FROM gcr.io/distroless/cc-debian12
COPY --from=builder /app/target/release/my-service /usr/local/bin/my-service
USER 10001:10001
ENTRYPOINT ["/usr/local/bin/my-service"]
```

Use `--locked` in CI to guarantee dependency resolution.

## Image Size Optimization

| Technique | Size impact | Notes |
|-----------|------------|-------|
| Distroless base | 100→12 MB | No shell, no package manager |
| `strip = true` in Cargo.toml | 12→8 MB | Removes debug symbols |
| `cargo chef` | Cached deps | Speeds up rebuilds significantly |
| `panic = "abort"` | 8→5 MB | Removes unwind tables |
| `lto = "fat"` | 5→4 MB | Link-time optimization (slower build) |
| UPX compression | 4→1.5 MB | Runtime decompression overhead |

### cargo-chef for Faster Rebuilds

```dockerfile
FROM rust:1.82-slim AS planner
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN cargo chef prepare --recipe-path recipe.json

FROM rust:1.82-slim AS builder
WORKDIR /app
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
COPY src ./src
RUN cargo build --release --locked

FROM gcr.io/distroless/cc-debian12
COPY --from=builder /app/target/release/my-service /usr/local/bin/my-service
USER 10001:10001
ENTRYPOINT ["/usr/local/bin/my-service"]
```

### Base Image Comparison

| Base image | Size | Security surface | Shell | Use case |
|------------|------|-----------------|-------|----------|
| `rust:1.82-slim` | ~800 MB | Large | Yes | Build stage only |
| `gcr.io/distroless/cc-debian12` | ~12 MB | Minimal | No | Production |
| `gcr.io/distroless/static-debian12` | ~5 MB | Minimal | No | Static binaries |
| `scratch` | ~2 MB | None | No | Fully static musl builds |
| `alpine:3.20` | ~7 MB | Small | Yes | When you need a shell |
| `ubuntu:22.04` | ~30 MB | Moderate | Yes | When you need apt packages |

## Docker Buildx / Caching

```dockerfile
# Use BuildKit cache mounts to avoid re-downloading crates
FROM rust:1.82-slim AS builder
WORKDIR /app

# Cache cargo registry
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release --locked
```

```yaml
# .github/workflows/container.yml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## Non-Root Container User

```dockerfile
# Option A: distroless (uses built-in non-root)
FROM gcr.io/distroless/cc-debian12
# Already runs as non-root

# Option B: alpine
FROM alpine:3.20
RUN addgroup -S app && adduser -S app -G app
USER app:app

# Option C: debian-based
FROM debian:bookworm-slim
RUN groupadd -r app && useradd -r -g app app
USER app:app
```

## Kubernetes Probes

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: my-service
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 10
            periodSeconds: 15
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            initialDelaySeconds: 3
            periodSeconds: 3
            failureThreshold: 10
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

Readiness should fail when dependencies are unavailable. Startup probe prevents premature liveness killing during slow initialization.

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-service
```

## Graceful Shutdown

Listen for `SIGTERM`, stop accepting new requests, drain in-flight work, then exit before `terminationGracePeriodSeconds`.

```rust
let server = axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal());
server.await?;
```

```yaml
# Ensure this is long enough for your drain
terminationGracePeriodSeconds: 30
```

### Drain Pattern for Worker Queues

```rust
async fn worker_loop(rx: tokio::sync::mpsc::Receiver<Job>) {
    let (mut rx, mut shutdown) = tokio::sync::mpsc::channel(100);
    // ...
    loop {
        tokio::select! {
            Some(job) = rx.recv() => { process(job).await; }
            _ = shutdown.recv() => {
                // Drain remaining in-flight, reject new
                while let Some(job) = rx.try_recv().ok() { process(job).await; }
                break;
            }
        }
    }
}
```

## Runtime Configuration

Use environment variables for deployment-specific settings. Never bake secrets into images.

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Config {
    #[serde(default = "default_host")]
    host: String,

    #[serde(default = "default_port")]
    port: u16,

    database_url: String,     // from env var DATABASE_URL
    redis_url: Option<String>,
}

fn config_from_env() -> Config {
    envy::from_env::<Config>().expect("missing env vars")
}
```

### Config Precedence

```
CLI flags  >  env vars  >  config file  >  compile-time defaults
```

## Secrets Management

```rust
// Good: read from env var injected by platform
let db_url = std::env::var("DATABASE_URL")
    .expect("DATABASE_URL must be set");

// Good: read from file mounted by Kubernetes Secret
let db_url = std::fs::read_to_string("/etc/secrets/database_url")
    .expect("secret not found");

// Bad: hardcoded in source
let db_url = "postgres://user:password@localhost/db";
```

Platform-specific secret sources:
- **Kubernetes**: Secret volumes mounted to `/etc/secrets/`
- **AWS**: Secrets Manager via SDK
- **Vault**: Agent sidecar injecting env vars
- **SOPS**: Encrypted config files decrypted at deploy time

## Health Signals

- **Liveness** (`/healthz`): process is alive (cheap, no dependencies).
- **Readiness** (`/readyz`): process can serve traffic (checks dependencies).
- **Startup** (`/startupz`): process completed migrations/cache warmup.

Keep probes cheap and bounded with timeouts.

```rust
use axum::{routing::get, Json, Router, http::StatusCode};
use std::sync::atomic::{AtomicBool, Ordering};

struct HealthState {
    ready: AtomicBool,
    started: AtomicBool,
}

async fn liveness() -> &'static str { "ok" }

async fn readiness(State(state): State<Arc<HealthState>>) -> Result<&'static str, StatusCode> {
    if state.ready.load(Ordering::Acquire) {
        Ok("ok")
    } else {
        Err(StatusCode::SERVICE_UNAVAILABLE)
    }
}

async fn startup(State(state): State<Arc<HealthState>>) -> Result<&'static str, StatusCode> {
    if state.started.load(Ordering::Acquire) {
        Ok("ok")
    } else {
        Err(StatusCode::SERVICE_UNAVAILABLE)
    }
}
```

## Deployment Strategies

| Strategy | Risk | Downtime | Complexity | Best for |
|----------|------|----------|------------|----------|
| Rolling update | Low | None | Minimal | Default |
| Blue-green | Very low | None | Moderate | Critical services |
| Canary | Low | None | High | Gradual rollout |
| Recreate | High | Yes | None | Dev/scratch |

### Blue-Green with Kubernetes

```yaml
# Green (current stable)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service-green
  labels:
    app: my-service
    track: green
---
# Blue (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service-blue
  labels:
    app: my-service
    track: blue
---
# Service switches selector between green and blue
apiVersion: v1
kind: Service
spec:
  selector:
    app: my-service
    track: green  # switch to "blue" after validation
```

## CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo test --locked

  docker:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.IMAGE_TAG }}

  deploy:
    needs: docker
    runs-on: ubuntu-latest
    steps:
      - uses: azure/setup-kubectl@v4
      - run: kubectl set image deployment/my-service app=${{ env.IMAGE_TAG }}
```

## Feature Flags in Deployments

```rust
// Use config-level feature flags, not compile-time
#[derive(Deserialize)]
struct Features {
    new_checkout_flow: bool,
}

async fn checkout(State(features): State<Arc<Features>>) -> impl IntoResponse {
    if features.new_checkout_flow {
        new_checkout().await
    } else {
        legacy_checkout().await
    }
}
```

Launch darkly, Unleash, or a simple config map toggle. Feature flags decouple deploy from release.

## Release Checklist

- `cargo test --locked`
- `cargo clippy --all-targets --all-features -- -D warnings`
- Container image scan passes (Trivy, Grype)
- Image size within threshold (target < 20 MB)
- Readiness/liveness endpoints tested
- Shutdown path tested (SIGTERM → drain → exit within grace period)
- Metrics dashboard and alerts exist
- Resource requests/limits set
- PDB configured for HA
- Rollback plan documented
