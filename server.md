> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# HTTP Server

**Module:** `viontin_framework::server`

TCP-based HTTP server with a path-matching router — zero external networking dependencies.

---

## Architecture

```
Server
  │
  ├── TcpListener (std::net)
  │
  └── Per-connection threads
        │
        ├── Parse HTTP/1.1 request (manual parser)
        ├── Router.match(method, path)
        ├── Route params extraction
        ├── Handler(request) → Response
        └── Write raw HTTP response to stream
```

---

## Server

```rust
pub struct Server {
    router: Arc<Router>,
}
```

### Methods

```rust
let server = Server::new(router);
server.run("127.0.0.1:3000")?;
```

Starts a TCP listener, accepts connections, and spawns a thread per connection.

---

## Router

```rust
pub struct Router {
    routes: Vec<(Method, String, Handler)>,
    not_found: Option<Handler>,
}
```

### Route Registration

```rust
Router::new()
    .get("/users", list_handler)
    .get("/users/:id", show_handler)
    .post("/users", create_handler)
    .any("/fallback", catch_all)
```

### Path Matching

Routes support `:param` syntax for dynamic segments:

| Pattern | Matches | Params |
|---------|---------|--------|
| `/users` | `/users` | — |
| `/users/:id` | `/users/42` | `id=42` |
| `/posts/:year/:slug` | `/posts/2024/hello` | `year=2024, slug=hello` |

Exact match precedence — no regex, no trie, linear scan.

### Not Found

```rust
Router::new()
    // custom 404 handler
    // default returns 404 with "404"
```

### Handler Type

```rust
pub type Handler = Arc<dyn Fn(Request) -> Response + Send + Sync>;
```

---

## Request Flow

```rust
fn handler(req: Request) -> Response {
    let name = req.param("name").unwrap_or("world");
    Response::html(format!("<h1>Hello, {}!</h1>", name))
}
```

### Connection Lifecycle

1. TCP connection accepted
2. HTTP request line parsed (`GET /path HTTP/1.1`)
3. Headers parsed
4. Body read (if `Content-Length` present)
5. `Router.handle(request)` → `Response`
6. Raw response written to stream

---

## Boot Integration

```rust
use viontin::boot;

fn main() {
    boot()
        .get("/", home)
        .get("/users/:id", show_user)
        .routes(|r| {
            r.get("/admin", admin_panel)
        })
        .serve("127.0.0.1:3000");
}
```

The `Boot::serve()` method:
1. Runs gem hooks and service providers
2. Dispatches CLI commands if arguments exist
3. Merges `RouteRegistry` handlers via `extend_from_registry()`
4. Attaches WebSocket routes
5. Starts the TCP server

---

## Complete Example

```rust
use viontin::prelude::*;

fn home(_req: Request) -> Response {
    Response::html("<h1>Home</h1>")
}

fn show_user(req: Request) -> Response {
    let id = req.param("id").unwrap_or("unknown");
    Response::json(&serde_json::json!({"id": id, "name": "Alice"})).unwrap()
}

fn create_user(req: Request) -> Response {
    let data: serde_json::Value = req.json().unwrap_or(serde_json::json!({}));
    Response::json(&data).unwrap().status(StatusCode::CREATED)
}

fn main() {
    Router::new()
        .get("/", Arc::new(home))
        .get("/users/:id", Arc::new(show_user))
        .post("/users", Arc::new(create_user));
}
```
