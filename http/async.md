# Async Server
> Last updated: 2026-05-25

**Module:** `viontin_framework::server_async`  
**Feature:** `async`

Tokio-based async HTTP server — alternative to the default synchronous thread-per-connection server.

---

## Enabling Async

```toml
[dependencies]
framework = { features = ["async"] }
```

This enables the `server_async` module which uses `tokio` under the hood.

---

## AsyncServer

```rust
use viontin_framework::server_async::AsyncServer;
use viontin_framework::server::Router;

#[tokio::main]
async fn main() {
    let router = Router::new()
        .get("/", Arc::new(|_| Response::html("Hello, async!")));

    let server = AsyncServer::new(router);
    server.run("127.0.0.1:3000").await.unwrap();
}
```

### Methods

```rust
let server = AsyncServer::new(router);
server.run("127.0.0.1:3000").await?;
```

Accepts the same `Router` as the sync server — all routing, middleware, and handler logic is shared.

---

## How it Works

| Aspect | Sync Server | Async Server |
|--------|-------------|--------------|
| **Runtime** | `std::thread` per connection | `tokio::spawn` per connection |
| **TCP** | `std::net::TcpListener` | `tokio::net::TcpListener` |
| **I/O** | blocking `read`/`write` | async `read`/`write` |
| **Feature gate** | always available | `async` feature required |
| **Router** | shared `Router` | same shared `Router` |

The async server uses the same `Router`, `Request`, `Response`, and `Middleware` types — no code changes needed when switching between sync and async.

---

## Full Boot Example

```rust
use std::sync::Arc;
use viontin::boot;
use viontin_framework::server::Router;
use viontin_framework::server_async::AsyncServer;

fn build_router() -> Router {
    Router::new()
        .get("/", Arc::new(|_| Response::html("<h1>Async!</h1>")))
        .get("/api/ping", Arc::new(|_| Response::json(&serde_json::json!({"ok": true})).unwrap()))
}

#[tokio::main]
async fn main() {
    let router = build_router().extend_from_registry();
    let server = AsyncServer::new(router);
    server.run("127.0.0.1:3000").await.unwrap();
}
```

---

## Comparison

| Criteria | Sync | Async |
|----------|------|-------|
| **Dependencies** | None (std only) | tokio |
| **Connection model** | 1 thread per connection | 1 task per connection |
| **Max connections** | Thread pool limited | Task pool (thousands) |
| **Compile time** | Fast | Slower (tokio) |
| **Complexity** | Minimal | Slightly more |
| **Use case** | Development, simple APIs | High concurrency, WebSocket |

---

## License

This module uses `tokio` (MIT License).