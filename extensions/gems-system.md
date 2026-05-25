# Gems — Developer Guide
> Last updated: 2026-05-25

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

Gems are plugins that extend Viontin at build time and runtime via Rust trait implementations. They can generate assets, register middleware, add CLI commands, and hook into the application lifecycle.

> **Planned:** WASM-based plugin loading for dynamically compiled gems.

---

## Available Gems

| Gem | Package | Purpose |
|-----|---------|---------|
| **TailwindCSS** | `tailwind` | Build-time CSS generation |
| **Inertia** | `inertia` | InertiaJS SPA bridge |

---

## Installing a Gem

Add the gem crate to your `Cargo.toml`:

```toml
[dependencies]
tailwind = { path = "../../products/gems/crates/tailwind" }
inertia = { path = "../../products/gems/crates/inertia" }
```

---

## Registering a Gem

Register gems via `boot().gem()`:

```rust
use viontin::boot;
use viontin_gem_tailwind::Tailwind;
use viontin_gem_inertia::Inertia;

fn main() {
    boot()
        .gem(Tailwind::load())
        .gem(Inertia::load().entry("resources/views/app.html"))
        .serve(":3000");
}
```

### What `.gem()` Does

When you call `.gem(gem)`:

1. The gem's `before_build()` hook is collected for later execution
2. If the gem implements `GemBinding`, its **middleware, providers, commands, and routes are automatically wired** — no extra `.middleware()` or `.provider()` calls needed
3. The gem is stored in the registry

### Example: Inertia Gem

```rust
boot()
    .gem(Inertia::load().entry("views/app.html"))  // middleware auto-registered via GemBinding
    .get("/", |_| inertia("Home", json!({})))
    .serve(":3000");
```

No `.middleware(Inertia::middleware())` needed — the binding handles it.

---

## Multiple Gems

```rust
boot()
    .gem(Tailwind::load())
    .gem(Inertia::load().entry("views/app.html"))
    .gem(MyCustomGem::load())
```

---

## Gem Lifecycle

During `.serve(":3000")` or `.run()`:

```
boot()
  ├── gem(GemA)               → register + auto-wire bindings
  ├── gem(GemB)               → register + auto-wire bindings
  ├── .run()
  │     ├── gems.before_build_all()    ← hooks execute here
  │     │     ├── Tailwind: generate CSS
  │     │     └── Inertia:  load root view template
  │     ├── app.run()
  │     ├── CLI dispatch (if args match a command) → exit
  │     └── entry(|ctx| { ... })       ← user callback
```

### before_build

Runs before the server starts. Used for:
- Asset generation (TailwindCSS, JS bundling)
- Template loading (Inertia root view)
- Configuration validation
- File system setup

### after_build

Runs after the server starts. Used for:
- Cleanup
- Logging
- Post-deployment notifications

---

## GemBinding — Automatic Wiring

Gems that need to register middleware, providers, commands, or routes can implement `GemBinding`. When you call `.gem()`, these are automatically wired by the `Boot` builder.

```rust
// Example: tailwind gem (no binding needed)
impl GemBinding for Tailwind {}  // empty — all defaults
```

```rust
// Example: inertia gem (registers middleware)
impl GemBinding for Inertia {
    fn gem_middlewares(&self) -> Vec<Box<dyn Middleware>> {
        vec![Box::new(InertiaMiddleware::new())]
    }
}
```

### What Gets Wired

| Binding Method | Boot Target | Description |
|---------------|-------------|-------------|
| `gem_middlewares()` | Global middleware chain | Applied to every route |
| `gem_providers()` | DI container (Application) | Register services at boot |
| `gem_commands()` | CLI kernel | Available via CLI |
| `gem_routes()` | HTTP router | Additional route configuration |

---

## API Reference

### GemBuilder — Constructor Contract

```rust
pub trait GemBuilder: Sized {
    fn load() -> Self;
}
```

All gems must implement `GemBuilder`. The `load()` method is the standard parameterless constructor. Optional configuration is done via builder chaining:

```rust
MyGem::load().option(value)
```

### GemFacade — Lifecycle Hooks

```rust
pub trait GemFacade: Debug + Send + Sync {
    fn meta(&self) -> &GemMeta;
    fn before_build(&self) -> Result<()> { Ok(()) }
    fn after_build(&self) -> Result<()> { Ok(()) }
}
```

| Method | When | Purpose |
|--------|------|---------|
| `meta()` | Required | Return gem identity metadata |
| `before_build()` | During `run()` | Asset generation, config validation, template loading |
| `after_build()` | After server start | Cleanup, logging, notifications |

### GemBinding — Automatic Wiring

```rust
pub trait GemBinding: GemFacade {
    fn gem_middlewares(&self) -> Vec<Box<dyn Middleware>> { vec![] }
    fn gem_providers(&self) -> Vec<Box<dyn ServiceProvider>> { vec![] }
    fn gem_commands(&self) -> Vec<Box<dyn Command>> { vec![] }
    fn gem_routes(&self) -> Option<fn(Router) -> Router> { None }
}
```

| Method | Auto-wired to | Purpose |
|--------|---------------|---------|
| `gem_middlewares()` | Global middleware chain | Intercept every request |
| `gem_providers()` | DI container | Register services |
| `gem_commands()` | CLI kernel | Add CLI commands |
| `gem_routes()` | HTTP router | Add static files or SPA fallback |

### GemRegistry

```rust
impl GemRegistry {
    pub fn register(&mut self, gem: impl GemFacade + 'static);
    pub fn remove(&mut self, name: &str);
    pub fn get(&self, name: &str) -> Option<&dyn GemFacade>;
    pub fn all(&self) -> Vec<&dyn GemFacade>;
    pub fn by_kind(&self, kind: GemKind) -> Vec<&dyn GemFacade>;
}
```

### GemMeta

```rust
pub struct GemMeta {
    pub name: &'static str,
    pub version: &'static str,
    pub description: &'static str,
    pub kind: GemKind,
    pub homepage: &'static str,
}
```

```rust
pub const META: GemMeta = GemMeta::new("my-gem", "0.1.0", "Description", GemKind::Integration);
```

---

## Complete Example

```rust
use viontin::boot;
use viontin_gem_tailwind::Tailwind;
use viontin_gem_inertia::Inertia;

fn main() {
    boot()
        .gem(Tailwind::load())                              // build-time CSS
        .gem(Inertia::load().entry("resources/views/app.html"))  // SPA bridge
        .get("/", |_| inertia("Home", json!({ "title": "Welcome" })))
        .serve(":3000");
}
```