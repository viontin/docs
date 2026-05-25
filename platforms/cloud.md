> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Cloud Services

**Platform type:** Cloud-native microservices, APIs, distributed systems

Viontin is built for the cloud from the ground up — health checks, graceful shutdown, service mesh patterns, and observability hooks.

## Architecture

```
Cloud Service
  |
  +-- HTTP Server (sync or async)
  +-- Health checks: /healthz (liveness), /readyz (readiness)
  +-- Graceful shutdown: SIGTERM -> drain -> stop
  +-- Middleware chain: CORS, Rate Limit, Request ID, Panic Recovery
  +-- Observability: logging, metrics (planned), tracing (planned)
  +-- Cloud storage: LocalStorage, MemoryStorage (S3 planned)
```

## Quick Start

```rust
use viontin::prelude::*;

fn main() {
    boot()
        .middleware(RequestId)
        .middleware(CorsMiddleware::permissive())
        .middleware(PanicRecovery)
        .get("/healthz", |_| Response::json(&serde_json::json!({"status":"ok"})))
        .serve("0.0.0.0:3000");
}
```

## Key Features

| Feature | Status | Description |
|---------|--------|-------------|
| Health checks | Built-in | /healthz liveness, /readyz readiness |
| Graceful shutdown | Built-in | SIGTERM/SIGINT drain and stop |
| CORS middleware | Built-in | CorsMiddleware permissive/origin |
| Request ID tracing | Built-in | RequestId middleware |
| Panic recovery | Built-in | PanicRecovery middleware |
| Rate limiting | Built-in | RateLimitMiddleware |
| Structured logging | Planned | JSON log output |
| Metrics | Planned | Prometheus /metrics endpoint |
| Tracing | Planned | OpenTelemetry spans |
| S3 storage | Planned | S3-compatible driver |
| Service discovery | Planned | DNS or Consul resolution |
