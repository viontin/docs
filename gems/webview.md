# Webview Gem

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

**Crate:** `viontin-gem-webview`  
**Gem Name:** `webview`  
**Kind:** `Integration`

Desktop webview integration using `wry` + `tao` — opens a native window with an HTML/CSS/JS frontend served by the embedded Viontin HTTP server.

---

## Architecture

```
Desktop Window (tao)
  │
  ├── WebView (wry)
  │     └── Loads http://127.0.0.1:{port}
  │           └── HTML/CSS/JS (React, Vue, Svelte, or plain)
  │
  └── IPC Channel (window.ipc.postMessage)
        │
        ▼
  Background Thread
    └── Viontin HTTP Server
          ├── Router
          ├── Middleware
          ├── API routes
          └── Static file serving
```

---

## Installation

```toml
[dependencies]
viontin-gem-webview = { path = "../../repos/gems/crates/viontin-gem-webview" }
```

### System Dependencies

**Linux:** Requires `libwebkit2gtk-4.1-dev`:

```bash
sudo apt install libwebkit2gtk-4.1-dev
```

**Windows:** WebView2 is bundled with Edge Chromium (included in Windows 11).

**macOS:** WebKit is built into the system.

---

## Usage

### Basic Setup

```rust
use viontin::boot;
use viontin_gem_webview::WebviewGem;

fn main() {
    boot()
        .gem(WebviewGem::load()
            .title("My App")
            .size(1024, 768)
        )
        .get("/", |_| {
            Response::html(r#"<h1>Hello from Desktop!</h1>"#)
        })
        .entry(|ctx| {
            viontin_gem_webview::launch(ctx.ws_server, ":3000");
        })
        .run();
}
```

### With SPA Frontend

```rust
use viontin::boot;
use viontin_gem_webview::WebviewGem;
use viontin_gem_inertia::{Inertia, inertia};

fn main() {
    boot()
        .gem(WebviewGem::load()
            .title("Dashboard")
            .size(1280, 800)
            .devtools(true)
        )
        .gem(Inertia::load().entry("resources/views/app.html"))
        .get("/", || inertia("Dashboard", json!({ "user": "Alice" })))
        .entry(|ctx| {
            viontin_gem_webview::launch(ctx.ws_server, ":3000");
        })
        .run();
}
```

### DevTools

Enable devtools during development:

```rust
.gem(WebviewGem::load()
    .title("My App")
    .devtools(true)
)
```

This opens the Chromium DevTools panel alongside the window.

---

## Lifecycle

```
.run()
  │
  ├── gems.before_build_all()
  │     └── WebviewGem: store config
  ├── app.run()
  ├── CLI dispatch (if args match) → exit
  │
  └── entry(|ctx| launch(ctx.ws_server, ":3000"))
        │
        ├── thread::spawn → HTTP server (background)
        ├── Create window + webview (main thread)
        └── Event loop → closes when window is closed
              └── Process exits → server thread terminates
```

---

## API Reference

### WebviewGem — Builder

| Method | Input | Description |
|--------|-------|-------------|
| `load()` | — | Create a new WebviewGem (implements `GemBuilder`) |
| `.title(name)` | `&str` | Set window title (default: `"Viontin App"`) |
| `.size(w, h)` | `u32, u32` | Set window size in pixels (default: `1024 x 768`) |
| `.devtools(bool)` | `bool` | Enable Chromium DevTools (default: `false`) |

### Launch

| Function | Input | Description |
|----------|-------|-------------|
| `launch(ws_server, addr)` | `WsServer, &str` | Start HTTP server in background + open desktop window |

The `addr` parameter follows Viontin server convention: `":3000"` auto-expands to `http://127.0.0.1:3000`.

---

## Communication: Rust ↔ JS

### JS → Rust (via IPC)

In your frontend JavaScript:

```javascript
window.ipc.postMessage(JSON.stringify({
    type: "command",
    name: "cache:clear"
}));
```

On the Rust side, the `with_ipc_handler` on the webview processes messages. Extend the `launch` function to handle custom commands:

```rust
// Future: custom IPC handling
let _webview = wry::WebViewBuilder::new()
    .with_url(&url)
    .with_ipc_handler(|request| {
        let msg = request.body();
        match msg {
            "ping" => println!("[webview] Ping from JS"),
            _ => println!("[webview] Unknown: {}", msg),
        }
    })
    .build(&window)
    .expect("Failed to create webview");
```

### Rust → JS

```rust
// Evaluate JavaScript in the webview at runtime
webview.evaluate_script("alert('Hello from Rust!')").ok();
```

---

## Complete Example

```rust
use viontin::boot;
use viontin_gem_webview::WebviewGem;

fn main() {
    boot()
        .gem(WebviewGem::load()
            .title("Viontin Desktop")
            .size(1280, 720)
        )
        .get("/", |_| {
            Response::html(include_str!("../../frontend/dist/index.html"))
        })
        .get("/api/health", |_| {
            Response::json(&serde_json::json!({ "status": "ok" }))
        })
        .entry(|ctx| {
            viontin_gem_webview::launch(ctx.ws_server, ":3000");
        })
        .run();
}
```

---

## License

MIT — same as Viontin.
