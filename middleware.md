# Middleware

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

## Built-in Middleware Ideas

The framework provides no built-in middleware — here are common ones to build:

| Middleware | Purpose |
|-----------|---------|
| Logger | Log request method, path, duration |
| CORS | Set CORS headers |
| Auth | Validate session/token, return 401 |
| CSRF | Validate CSRF token on state-changing methods |
| Inertia | InertiaJS protocol bridge |
| Static Files | Serve files from directory |
| Compress | Gzip/Brotli response compression |
| Rate Limit | Rate limiting per IP/user |

---

## Complete Example

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct CorsMiddleware;

impl Middleware for CorsMiddleware {
    fn handle(&self, req: &mut Request, next: &dyn Fn(&mut Request) -> Response) -> Response {
        let mut res = next(req);
        res.headers.set("Access-Control-Allow-Origin", "*");
        res.headers.set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
        res
    }
}

fn main() {
    boot()
        .middleware(LoggerMiddleware)
        .middleware(CorsMiddleware)
        .get("/api/users", |_| {
            Response::json(&serde_json::json!([{"id": 1, "name": "Alice"}])).unwrap()
        })
        .serve("127.0.0.1:3000");
}
```
