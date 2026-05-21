# Gems — Developer Guide

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

Gems are plugins that extend Viontin at build time and runtime. They can generate assets, register middleware, add CLI commands, and hook into the application lifecycle.

---

## Available Gems

| Gem | Package | Purpose |
|-----|---------|---------|
| **TailwindCSS** | `viontin-gem-tailwind` | Build-time CSS generation |
| **Inertia** | `viontin-gem-inertia` | InertiaJS SPA bridge |

---

## Installing a Gem

Add the gem crate to your `Cargo.toml`:

```toml
[dependencies]
viontin-gem-tailwind = { path = "../../repos/gems/crates/viontin-gem-tailwind" }
viontin-gem-inertia = { path = "../../repos/gems/crates/viontin-gem-inertia" }
```

---

## Registering a Gem

Register gems via `boot().gem()`:

```rust
use viontin::boot;

fn main() {
    boot()
        .gem(viontin_gem_tailwind::Gem)
        .gem(viontin_gem_inertia::Inertia::new("resources/views/app.html"))
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
    .gem(Inertia::new("views/app.html"))  // middleware auto-registered via GemBinding
    .get("/", |_| inertia("Home", json!({})))
    .serve(":3000");
```

No `.middleware(Inertia::middleware())` needed — the binding handles it.

---

## Multiple Gems

```rust
boot()
    .gem(TailwindGem)
    .gem(Inertia::new("views/app.html"))
    .gem(MyCustomGem)
```

Or using the plural form:

```rust
boot()
    .withGems(vec![
        Box::new(TailwindGem),
        Box::new(Inertia::new("views/app.html")),
    ])
```

---

## Gem Lifecycle

During `boot().serve(":3000")`:

```
boot()
  ├── gem(GemA)               → register + auto-wire bindings
  ├── gem(GemB)               → register + auto-wire bindings
  ├── .serve(":3000")
  │     ├── gems.before_build_all()    ← hooks execute here
  │     │     ├── Tailwind: generate CSS
  │     │     └── Inertia:  load root view template
  │     ├── app.run()
  │     └── server starts
  └── gems.after_build_all()           ← cleanup hooks here
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
impl GemBinding for TailwindGem {}  // empty — all defaults
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

## Managing Gems at Runtime

```rust
// Access the gem registry
// Gems are registered automatically by boot(), but you can also access them
// through the framework at runtime if needed.
```

---

## Complete Example

```rust
use viontin::boot;
use viontin_gem_tailwind::Gem as TailwindGem;
use viontin_gem_inertia::Inertia;

fn main() {
    boot()
        .gem(TailwindGem)                        // build-time CSS
        .gem(Inertia::new("resources/views/app.html"))  // SPA bridge
        .get("/", |_| inertia("Home", json!({ "title": "Welcome" })))
        .serve(":3000");
}
```
