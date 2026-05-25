> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Building Web Applications

---

Viontin includes a full-featured HTTP server with routing, middleware, WebSocket support, sessions, CSRF protection, and more — all with zero external HTTP dependencies.

Two UI strategies are supported out of the box: **InertiaJS** for server-driven SPAs with your favorite frontend framework, and **Webview** for native desktop applications.

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

## 11. UI Strategies

Viontin supports two approaches for building user interfaces, each suited to different use cases.

### 11.1 InertiaJS — Server-Driven SPA

[InertiaJS](https://inertiajs.com/) lets you build single-page applications using classic server-side routing with React, Vue, or Svelte frontends — without building an API.

**How it works:**
1. First request: server renders full HTML with embedded `data-page` JSON
2. Subsequent navigations: Inertia client makes XHR requests, server returns JSON
3. No API endpoints needed — routes return page components directly

**Setup:**

```toml
[dependencies]
inertia = { path = "../../products/gems/crates/inertia" }
```

```rust
use viontin::boot;
use viontin::gem::GemBuilder;
use viontin_gem_inertia::{Inertia, inertia};

fn main() {
    boot()
        .gem(Inertia::load().entry("resources/views/app.html"))
        .get("/", |_| inertia("Home", json!({ "title": "Welcome" })))
        .serve(":3000");
}
```

**Root view template** (`resources/views/app.html`):

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>My App</title>
    <link rel="stylesheet" href="/assets/app.css">
</head>
<body>
    <div id="app" data-page="{{data-page}}"></div>
    <script src="/assets/app.js" defer></script>
</body>
</html>
```

The `{{data-page}}` placeholder is replaced at compile time with the Inertia page JSON on full-page loads.

**Request flow:**

```
Browser                           Viontin
  │                                  │
  ├── Initial visit ────────────────►│
  │                                  ├── Render HTML + data-page JSON
  │◄─────────────────────────────────┤
  │                                  │
  ├── SPA navigation ──────────────►│
  │   X-Inertia: true                │
  │                                  ├── Return JSON
  │◄─────────────────────────────────┤  { component, props, url, version }
```

### 11.2 Webview — Native Desktop

The webview gem wraps your Viontin HTTP server in a native desktop window using `wry` + `tao`. The frontend can be any HTML/CSS/JS application served by the embedded server.

**Setup:**

```toml
[dependencies]
webview = { path = "../../products/gems/crates/webview" }
```

**System dependencies (Linux):**
```bash
sudo apt install libwebkit2gtk-4.1-dev libsoup-3.0-dev
```

**Basic usage:**

```rust
use viontin::boot;
use viontin_gem_webview::WebviewGem;

fn main() {
    boot()
        .gem(WebviewGem::load().title("My App").size(1024, 768))
        .get("/", |_| Response::html("<h1>Hello from Desktop!</h1>"))
        .entry(|ctx| {
            viontin_gem_webview::launch(ctx.ws_server, ":3000");
        })
        .run();
}
```

**Rust ↔ JavaScript communication:**

Send messages from JavaScript:

```javascript
window.ipc.postMessage(JSON.stringify({
    type: "command",
    payload: "cache:clear"
}));
```

Handle them in Rust:

```rust
use viontin_gem_webview::launch_with_ipc;

launch_with_ipc(ws_server, ":3000", Some(Box::new(|msg| {
    println!("[webview] JS says: {}", msg);
})));
```

### 11.3 Choosing a Strategy

| Criteria | InertiaJS | Webview |
|----------|-----------|---------|
| **Frontend tech** | React, Vue, Svelte | Any (HTML, JS, or WASM) |
| **Rendering** | Server-driven SPA | Client-side |
| **Desktop support** | Via browser | Native window |
| **API required** | No | Optional |
| **Mobile support** | Responsive web | Desktop only |
| **Setup complexity** | Low (gem + npm) | Medium (system deps) |

**Recommendation:** Use **InertiaJS** for web applications where you want a modern SPA experience without building a separate API. Use **Webview** for desktop applications that need native window management.

---

## 12. Complete Example

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