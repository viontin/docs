> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Building Web Applications

---

Viontin includes a full-featured HTTP server with routing, middleware, WebSocket support, sessions, CSRF protection, and more — all with zero external HTTP dependencies.

---

## 1. Routing

### Basic Routes

```rust
use viontin::prelude::*;

fn main() {
    boot()
        .get("/", home)
        .get("/users", list_users)
        .get("/users/:id", show_user)
        .post("/users", create_user)
        .any("/fallback", catch_all)
        .serve("127.0.0.1:3000");
}

fn home(_req: Request) -> Response {
    Response::html("<h1>Home</h1>")
}

fn show_user(req: Request) -> Response {
    let id = req.param("id").unwrap_or("unknown");
    Response::html(format!("<h1>User: {}</h1>", id))
}
```

### Route Parameters

Parameters are prefixed with `:` in the path pattern and extracted via `req.param()`:

```rust
.get("/posts/:year/:slug", |req| {
    let year = req.param("year").unwrap();
    let slug = req.param("slug").unwrap();
    // ...
})
```

### Group Routes via Closure

```rust
boot()
    .routes(|router| {
        router
            .get("/admin", admin_dashboard)
            .get("/admin/users", admin_users)
            .post("/admin/users", admin_create_user)
    })
```

### Route Registry (Global)

For larger applications, routes can be registered globally and merged at boot:

```rust
// routes/mod.rs
use viontin::route;

pub fn register() {
    route::get("/", crate::handlers::home);
    route::post("/contact", crate::handlers::contact);
}

// main.rs
fn main() {
    crate::routes::register();
    boot()
        .routes(|r| r.extend_from_registry())
        .serve("127.0.0.1:3000");
}
```

---

## 2. Request & Response

### Request

```rust
fn handler(req: Request) -> Response {
    req.method();           // Method::GET
    req.path();             // "/users/42"
    req.param("id");        // Some("42")
    req.header("Content-Type");  // Some("application/json")
    req.headers();          // &Headers
    req.body();             // &[u8]
    req.cookie("session");  // Option<String>

    // Query parameters
    let page = req.query("page").unwrap_or("1");
}
```

### Response

```rust
// HTML response
Response::html("<h1>Hello</h1>")

// JSON response
Response::json(r#"{"status":"ok"}"#)

// Plain text
Response::text("OK")

// With status code
Response::html("<h1>Not Found</h1>").status(StatusCode::NOT_FOUND)

// With custom headers
Response::html("content")
    .header("X-Custom", "value")

// With cookie
Response::html("content")
    .cookie("session", "abc123")
```

### Status Codes

```rust
StatusCode::OK                // 200
StatusCode::CREATED           // 201
StatusCode::NO_CONTENT        // 204
StatusCode::MOVED             // 301
StatusCode::FOUND             // 302
StatusCode::BAD_REQUEST       // 400
StatusCode::UNAUTHORIZED      // 401
StatusCode::NOT_FOUND         // 404
StatusCode::SERVER_ERROR      // 500
```

---

## 3. Static Files & Templates

### Content Embedding Macros

```rust
use viontin::{html, md, js, ts};

// Embed at compile time
let page = html!("pages/index.html");
let doc  = md!("docs/guide.md");
let script = js!("assets/app.js");
```

### Serve Static Files

Use the `fs` module for runtime file serving:

```rust
use viontin::fs;

fn serve_asset(req: Request) -> Response {
    let path = req.param("path").unwrap();
    match fs::read(format!("public/{}", path)) {
        Ok(content) => Response::text(content),
        Err(_) => Response::html("<h1>Not Found</h1>").status(StatusCode::NOT_FOUND),
    }
}
```

---

## 4. WebSocket

```rust
use viontin::prelude::*;
use viontin::ws::{Message, WebSocketHandler, WebSocketConfig};

struct ChatHandler;

impl WebSocketHandler for ChatHandler {
    fn on_message(&self, msg: Message) -> Option<Message> {
        println!("Received: {:?}", msg);
        Some(Message::Text(format!("Echo: {:?}", msg)))
    }
}

fn main() {
    boot()
        .ws("/chat", ChatHandler)
        .ws_with_config("/secure-chat", WebSocketConfig { max_message_size: 65536 }, SecureChatHandler)
        .serve("127.0.0.1:3000");
}
```

---

## 5. Sessions

```rust
use viontin::prelude::*;

// In-memory sessions (development)
let session = Session::memory();

// File-backed sessions (production)
let session = Session::file("storage/sessions/");

// In a handler
fn login(req: Request) -> Response {
    let mut session = Session::memory();
    session.set("user_id", "42");
    session.save();

    Response::html("Logged in")
        .cookie("session", session.id())
}
```

---

## 6. CSRF Protection

```rust
use viontin::prelude::*;

let csrf = CsrfManager::new(CsrfConfig::default());

// Generate a token
let token = csrf.token("session-id");

// Validate a token
if csrf.validate("session-id", &token) {
    // proceed
}
```

---

## 7. Caching

```rust
use viontin::prelude::*;

// Memory cache (fastest, ephemeral)
let cache = Cache::memory();

// File cache (persistent across restarts)
let cache = Cache::file("storage/cache/");

// Null cache (no-op for testing)
let cache = Cache::null();

cache.set("users_count", "1500", 3600); // TTL: 1 hour
let count = cache.get("users_count");   // Some("1500")
cache.has("users_count");               // true
cache.delete("users_count");            // remove
cache.clear();                          // remove all
```

---

## 8. Middleware via Service Providers

```rust
use viontin::prelude::*;

struct LogRequests;

impl ServiceProvider for LogRequests {
    fn register(&self, app: &mut Application) {
        // Register services
    }

    fn boot(&self, app: &Application) {
        let logger = app.resolve::<Logger>().unwrap();
        logger.info("Request middleware initialized");
    }
}

fn main() {
    boot()
        .provider(LogRequests)
        .get("/", handler)
        .serve("127.0.0.1:3000");
}
```

---

## 9. Configuration

```rust
use viontin::prelude::*;

// config/config.json
// { "app": { "name": "MyApp", "debug": true } }

fn handler(_req: Request) -> Response {
    let name = config("app.name").unwrap_or("Viontin");
    let debug = config("app.debug").unwrap_or("false");

    Response::html(format!("<h1>{} (debug: {})</h1>", name, debug))
}
```

Environment-specific overrides:

```
config/
├── config.json          # base config
├── config.dev.json      # overrides for dev environment
├── config.prod.json     # overrides for production
└── config.testing.json  # overrides for test suite
```

---

## 10. Error Handling

```rust
fn handler(_req: Request) -> Response {
    // Returns a 404 page
    Response::html("<h1>Not Found</h1>").status(StatusCode::NOT_FOUND)

    // Returns a 500 page (with debug info in dev mode)
    // FrameworkError::Internal(msg) → 500
}
```

---

## 11. Complete Example

```rust
use viontin::prelude::*;

fn main() {
    boot()
        .get("/", |_| Response::html(
            r#"<h1>My App</h1><a href="/users">Users</a>"#
        ))
        .get("/users", list_users)
        .get("/users/:id", show_user)
        .serve("127.0.0.1:3000");
}

fn list_users(_req: Request) -> Response {
    // In a real app, fetch from database
    Response::html("<h1>Users</h1><ul><li>Alice</li><li>Bob</li></ul>")
}

fn show_user(req: Request) -> Response {
    let id = req.param("id").unwrap_or("unknown");
    Response::html(format!("<h1>User: {}</h1>", id))
}
```
