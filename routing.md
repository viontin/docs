> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Routing

**Module:** `viontin_framework::route`

Global route metadata registry — track, inspect, and manage route definitions across your application.

---

## RouteRegistry

```rust
pub struct RouteRegistry {
    routes: HashMap<(RouteMethod, String), RouteAndSource>,
    finalized: bool,
}
```

Global singleton that tracks all routes, their handler names, and where they were defined. Routes can't be registered after finalization.

---

## RouteMethod

```rust
pub enum RouteMethod { Get, Post, Put, Patch, Delete, Head, Options, }
```

---

## RouteDefinition

```rust
pub struct RouteDefinition {
    pub method: RouteMethod,
    pub path: String,
    pub handler_name: String,
    pub source: String,    // file:line where registered
}
```

---

## Registering Routes

```rust
use viontin::route;

// Explicit
route::register(RouteMethod::Get, "/users", "UserController::index", "src/routes/users.rs:5");

// Shorthand
route::get("/", "HomeController::index", "src/routes/web.rs:10");
route::post("/submit", "FormController::submit", "src/routes/web.rs:11");
route::put("/users/:id", "UserController::update", "src/routes/web.rs:12");
route::delete("/users/:id", "UserController::destroy", "src/routes/web.rs:13");
```

### Route Conflict Detection

```rust
route::get("/api/users", "handler_a", "routes/a.rs:5");
route::get("/api/users", "handler_b", "routes/b.rs:5");
// Panic:
//   Route conflict: GET /api/users
//     Defined in: routes/a.rs:5
//     Conflict at: routes/b.rs:5
//   Resolution: remove one definition or use Route::remove() first.
```

### Removing Routes

```rust
route::remove(RouteMethod::Get, "/old-route");
```

### Checking Routes

```rust
route::has(&RouteMethod::Get, "/users");  // bool
route::all();                              // Vec<RouteDefinition>
```

### Finalizing

```rust
route::finalize();  // prevent further registrations
```

---

## Handler Registration

Separate from metadata — stores actual handler functions for the HTTP server:

```rust
use viontin::route;
use viontin::http::Method;

route::register_handler(Method::Get, "/users", Arc::new(users_handler));
route::register_handler(Method::Post, "/users", Arc::new(create_user_handler));
```

Retrieve all handlers for the server:

```rust
route::take_handlers();  // Vec<(Method, String, Handler)>
```

Used internally by `Boot::serve()` via `router.extend_from_registry()`.

---

## RouteProvider

Service provider that registers routes from the registry:

```rust
use viontin::app::RouteProvider;

boot()
    .provider(RouteProvider)
        .run_with(|_ctx| {});
```

---

## Complete Example

```rust
use viontin::route;
use viontin::http::Method;

// routes/web.rs
pub fn register() {
    route::get("/", "home", "routes/web.rs:5");
    route::get("/users", "users_index", "routes/web.rs:6");
    route::post("/users", "users_create", "routes/web.rs:7");
    route::get("/users/:id", "users_show", "routes/web.rs:8");
}

// main.rs
use viontin::boot;

fn main() {
    crate::routes::register();

    boot()
        .routes(|router| router.extend_from_registry())
        .serve("127.0.0.1:3000");
}
```
