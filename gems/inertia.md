# InertiaJS Gem

**Crate:** `viontin-gem-inertia`  
**Gem Name:** `inertia`  
**Kind:** `Integration`

Server-side adapter for [InertiaJS](https://inertiajs.com/) — build single-page apps using classic server-side routing with React, Vue, or Svelte frontends.

---

## Architecture

```
Browser                          Viontin
  │                                │
  ├── Initial visit ──────────────►│
  │                                ├── No X-Inertia header
  │                                ├── Render HTML + data-page JSON
  │◄───────────────────────────────┤
  │                                │
  ├── SPA navigation ────────────►│
  │   X-Inertia: true              │
  │                                ├── Return JSON
  │◄───────────────────────────────┤  { component, props, url, version }
  │                                │
  ├── Form submit ────────────────►│
  │                                ├── Validation fails
  │◄───────────────────────────────┤  409 + X-Inertia-Location
  │   (client re-fetches)          │
```

---

## Installation

```toml
[dependencies]
viontin-gem-inertia = { path = "../../repos/gems/crates/viontin-gem-inertia" }
```

### Client-side

Install the InertiaJS client adapter of your choice:

```bash
npm install @inertiajs/inertia
npm install @inertiajs/inertia-react   # for React
npm install @inertiajs/inertia-vue     # for Vue
npm install @inertiajs/inertia-svelte  # for Svelte
```

---

## Usage

### Basic Setup

```rust
use viontin::boot;
use viontin_gem_inertia::{Inertia, inertia};

fn main() {
    boot()
        .gem(Inertia::new("resources/views/app.html"))  // middleware auto-wired via GemBinding
        .get("/", |_| inertia("Home", json!({ "title": "Welcome" })))
        .get("/users", |_| inertia("Users/Index", json!({ "users": users })))
        .serve(":3000");
}
```

> **Note:** The middleware is registered automatically via `GemBinding` — no need to call `.middleware()` manually.

### Root View Template

Create `resources/views/app.html` with `{{data-page}}` where the page data will be injected:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>My App</title>
    <link rel="stylesheet" href="/assets/app.css">
</head>
<body>
    <div id="app" data-page="{{data-page}}"></div>
    <script src="/assets/app.js" defer></script>
</body>
</html>
```

The `{{data-page}}` placeholder will be replaced with the JSON page data on full-page loads.

---

## Inertia Gem

```rust
use viontin_gem_inertia::Inertia;

Inertia::new("resources/views/app.html")
```

The gem, during `before_build()`:
1. Reads the root view template from the file path
2. Stores it in global config for runtime use by the middleware

---

## InertiaPage Responses

### Basic

```rust
use viontin_gem_inertia::inertia;

inertia("Home", json!({ "title": "Welcome" }))
```

### With Additional Props

```rust
inertia("Users/Profile", json!({ "id": 42 }))
    .with(json!({ "role": "admin" }))
```

### Custom URL

```rust
inertia("Dashboard", data).url("/dashboard")
```

### Custom Status Code

```rust
inertia("Errors/NotFound", json!({})).status(404)
```

### Partial Reload (only requested props)

```rust
inertia("Dashboard", json!({ "users": users, "stats": stats }))
    .only(&["stats"])
```

Only `stats` will be returned when the client requests a partial reload.

### Redirect

```rust
InertiaPage::redirect("/login")
InertiaPage::back()
```

---

## Shared Props

Props available on every page (e.g., current user, flash messages):

```rust
use viontin_gem_inertia::share;

share("user", json!({ "id": 1, "name": "Alice" }));
share("flash", json!({ "success": "Saved!" }));
```

### Managing Shared Props

```rust
share("key", value);
unshare("key");
flush_shared();  // remove all shared props
```

---

## Asset Versioning

When assets change, Inertia forces a full page reload. Set a version:

```rust
use viontin_gem_inertia::set_version;

set_version("v2");
```

Default: `"1.0"`.

---

## InertiaMiddleware

```rust
Inertia::middleware()
    .with_root_view("views/app.html")
```

The middleware:

| Condition | Behavior |
|-----------|----------|
| `X-Inertia` header present | Returns JSON `{ component, props, url, version }` |
| No `X-Inertia` header | Renders HTML with embedded `data-page` JSON |
| Redirect (3xx) + `X-Inertia` | Returns 200 with `X-Inertia-Location` |
| Partial reload headers | Filters props to only requested keys |

---

## Complete Example

```rust
use viontin::boot;
use viontin_gem_inertia::{Inertia, inertia, share, set_version};

fn main() {
    boot()
        .gem(Inertia::new("resources/views/app.html"))
        .middleware(Inertia::middleware())
        .get("/", |_| {
            share("user", json!({ "id": 1, "name": "Alice" }));
            inertia("Home", json!({ "title": "Dashboard" }))
        })
        .get("/users", |_| {
            let users = vec![
                json!({"id": 1, "name": "Alice"}),
                json!({"id": 2, "name": "Bob"}),
            ];
            inertia("Users/Index", json!({ "users": users }))
        })
        .get("/users/create", |_| {
            inertia("Users/Create", json!({}))
        })
        .post("/users", |req| {
            // validate, save...
            InertiaPage::redirect("/users")
        })
        .serve(":3000");
}
```
