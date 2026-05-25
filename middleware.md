# Middleware
> Last updated: 2026-05-25

**Module:** `viontin_framework::middleware`

Middleware provides a request pipeline — intercept requests before they reach route handlers.

---

## Architecture

```
Request
  │
  ▼
MiddlewareChain
  │
  ├── Middleware 1 (e.g., Logger)
  │     └── calls next(req)
  ├── Middleware 2 (e.g., Auth)
  │     └── calls next(req)
  ├── Middleware 3 (e.g., CORS)
  │     └── calls next(req)
  │
  ▼
Route Handler → Response
```

Each middleware can:
- Execute code before/after the handler
- Modify the request
- Short-circuit by returning a response without calling `next`
- Modify the response before returning

---

## Middleware Trait

```rust
pub trait Middleware: Debug + Send + Sync {
    fn handle(&self, req: &mut Request, next: &dyn Fn(&mut Request) -> Response) -> Response;
}
```

### Example

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct LoggerMiddleware;

impl Middleware for LoggerMiddleware {
    fn handle(&self, req: &mut Request, next: &dyn Fn(&mut Request) -> Response) -> Response {
        println!("[{}] {}", req.method, req.uri.path);
        let start = std::time::Instant::now();
        let res = next(req);
        let dur = start.elapsed();
        println!("[{}] -> {} ({:?})", req.method, res.status, dur);
        res
    }
}
```

---

## MiddlewareChain

Ordered chain that applies middlewares in registration order.

### Methods

```rust
let mut chain = MiddlewareChain::new();

chain.add(LoggerMiddleware);
chain.add(Box::new(AuthMiddleware));    // via add_dyn for boxed
chain.add_fn(|req, next| {
    // closure middleware
    next(req)
});

chain.is_empty();  // bool
chain.len();       // usize
```

### apply

```rust
let response = chain.apply(&mut request, |req| {
    // route handler
    handler(std::mem::take(req))
});
```

---

## Closure Middleware

For simple middleware without a full struct:

```rust
chain.add_fn(|req, next| {
    println!("Before: {}", req.uri.path);
    let res = next(req);
    println!("After: {} -> {}", req.uri.path, res.status);
    res
});
```

---

## Registration

### Via Router (per-route)

```rust
use viontin::prelude::*;

let mut mw = MiddlewareChain::new();
mw.add(AuthMiddleware);

Router::new()
    .get_with("/admin", Arc::new(admin_handler), mw)
    .post_with("/admin", Arc::new(admin_post), mw)
```

### Via Router (global)

```rust
Router::new()
    .with_global_middleware(chain)
```

### Via Boot (global — recommended)

```rust
use viontin::boot;

boot()
    .middleware(LoggerMiddleware)         // single
    .withMiddlewares(vec![                // group
        Box::new(CorsMiddleware),
        Box::new(AuthMiddleware),
    ])
    .serve(":3000");
```

---

## Built-in Middleware

| Middleware | Constructor | Purpose |
|-----------|-------------|---------|
| `CorsMiddleware` | `CorsMiddleware::permissive()` or `CorsMiddleware::origin("https://app.com")` | Cross-Origin Resource Sharing — configure allowed origins, methods, headers, credentials. Handles OPTIONS preflight automatically. |
| `PanicRecovery` | `PanicRecovery` | Catch panics from downstream handlers, return 500 instead of crashing. |
| `RequestId` | `RequestId` | Attach unique `X-Request-Id` to every request/response. |
| `RateLimitMiddleware` | `RateLimitMiddleware::new("api", 100, 60)` | Token bucket rate limiting per key prefix. Returns 429 when exceeded. |

### CorsMiddleware Example

```rust
use viontin::prelude::*;

fn main() {
    boot()
        .middleware(CorsMiddleware::permissive())          // allow all origins
        .get("/api/users", |_| Response::json(&users))
        .serve("127.0.0.1:3000");
}
```

Or restrict to a specific origin:

```rust
boot()
    .middleware(CorsMiddleware::origin("https://myapp.com")
        .with_allowed_methods(&["GET", "POST"])
        .with_credentials(true))
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    boot()
        .middleware(RequestId)
        .middleware(CorsMiddleware::permissive())
        .middleware(PanicRecovery)
        .get("/api/health", |_| Response::json(&serde_json::json!({"status": "ok"})))
        .serve("127.0.0.1:3000");
}
```