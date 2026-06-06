---
name: domain-cloud-native
description: "Use when building cloud-native apps. Keywords: kubernetes, k8s, docker, container, grpc, tonic, microservice, service mesh, observability, tracing, metrics, health check, cloud, deployment, 云原生, 微服务, 容器"
user-invocable: false
---

# Cloud-Native Domain

> **Layer 3: Domain Constraints**

## Domain Constraints → Design Implications

| Domain Rule | Design Constraint | Rust Implication |
|-------------|-------------------|------------------|
| 12-Factor | Config from env | Environment-based config |
| Observability | Metrics + traces | tracing + opentelemetry |
| Health checks | Liveness/readiness | Dedicated endpoints |
| Graceful shutdown | Clean termination | Signal handling |
| Horizontal scale | Stateless design | No local state |
| Container-friendly | Small binaries | Release optimization |

---

## Critical Constraints

### Stateless Design

```
RULE: No local persistent state
WHY: Pods can be killed/rescheduled anytime
RUST: External state (Redis, DB), no static mut
```

### Graceful Shutdown

```
RULE: Handle SIGTERM, drain connections
WHY: Zero-downtime deployments
RUST: tokio::signal + graceful shutdown
```

### Observability

```
RULE: Every request must be traceable
WHY: Debugging distributed systems
RUST: tracing spans, opentelemetry export
```

---

## Trace Down ↓

From constraints to design (Layer 2):

```
"Need distributed tracing"
    ↓ m12-lifecycle: Span lifecycle
    ↓ tracing + opentelemetry

"Need graceful shutdown"
    ↓ m07-concurrency: Signal handling
    ↓ m12-lifecycle: Connection draining

"Need health checks"
    ↓ domain-web: HTTP endpoints
    ↓ m06-error-handling: Health status
```

---

## Key Crates

| Purpose | Crate |
|---------|-------|
| gRPC | tonic |
| Kubernetes | kube, kube-runtime |
| Docker | bollard |
| Tracing | tracing, opentelemetry |
| Metrics | prometheus, metrics |
| Config | config, figment |
| Health | HTTP endpoints |

## Design Patterns

| Pattern | Purpose | Implementation |
|---------|---------|----------------|
| gRPC services | Service mesh | tonic + tower |
| K8s operators | Custom resources | kube-runtime Controller |
| Observability | Debugging | tracing + OTEL |
| Health checks | Orchestration | `/health`, `/ready` |
| Config | 12-factor | Env vars + secrets |

## Code Pattern: Graceful Shutdown

```rust
use tokio::signal;

async fn run_server() -> anyhow::Result<()> {
    let app = Router::new()
        .route("/health", get(health))
        .route("/ready", get(ready));

    let addr = SocketAddr::from(([0, 0, 0, 0], 8080));

    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    Ok(())
}

async fn shutdown_signal() {
    signal::ctrl_c().await.expect("failed to listen for ctrl+c");
    tracing::info!("shutdown signal received");
}
```

## Code Pattern: gRPC Service-to-Actor Adapter

The `tonic` row above is the headline crate, but the decision that matters for a *stateful* gRPC service is how the RPC handler talks to your state. Don't put state behind `Arc<Mutex<_>>` inside the service struct — make the service a thin translator that forwards each RPC to a single state-owning actor over a `oneshot` reply channel. State mutations then serialize through one task, eliminating lock contention and async data races.

```rust
// crates/.../driver/server.rs — one RPC, verbatim shape (LakeSail — sail-execution/src/driver/server.rs:29-58)
async fn register_worker(
    &self,
    request: Request<RegisterWorkerRequest>,
) -> Result<Response<RegisterWorkerResponse>, Status> {
    let request = request.into_inner();
    debug!("{request:?}");
    let RegisterWorkerRequest { worker_id, host, port } = request;
    let (tx, rx) = oneshot::channel();
    let event = DriverEvent::RegisterWorker {
        worker_id: WorkerId::from(worker_id), host, port, result: tx,
    };
    self.handle.send(event).await.map_err(ExecutionError::from)?;
    rx.await.map_err(ExecutionError::from)??;
    Ok(Response::new(RegisterWorkerResponse {}))
}
```

**Why:** the service *is* the actor; the impl is a pure RPC↔event translator. The two `?` operators turn `oneshot::RecvError` and the inner result into `Status` cleanly, and the `debug!("{request:?}")` book-end gives per-RPC tracing for free (ties into the Observability constraint above).

**Gotcha:** the actor MUST respond — if it drops `tx` (panics/stops), `rx.await` yields `RecvError` and the client sees an opaque error. Add explicit "actor stopped" messaging for clean shutdown.

See **m07-concurrency** for the actor loop itself and **m12-lifecycle** for shutting it down.

## Health Check Pattern

```rust
async fn health() -> StatusCode {
    StatusCode::OK
}

async fn ready(State(db): State<Arc<DbPool>>) -> StatusCode {
    match db.ping().await {
        Ok(_) => StatusCode::OK,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE,
    }
}
```

---

## Common Mistakes

| Mistake | Domain Violation | Fix |
|---------|-----------------|-----|
| Local file state | Not stateless | External storage |
| No SIGTERM handling | Hard kills | Graceful shutdown |
| No tracing | Can't debug | tracing spans |
| Static config | Not 12-factor | Env vars |

---

## Trace to Layer 1

| Constraint | Layer 2 Pattern | Layer 1 Implementation |
|------------|-----------------|------------------------|
| Stateless | External state | Arc<Client> for external |
| Graceful shutdown | Signal handling | tokio::signal |
| Tracing | Span lifecycle | tracing + OTEL |
| Health checks | HTTP endpoints | Dedicated routes |

---

## Related Skills

| When | See |
|------|-----|
| Async patterns | m07-concurrency |
| HTTP endpoints | domain-web |
| Error handling | m13-domain-error |
| Resource lifecycle | m12-lifecycle |
