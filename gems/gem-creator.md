# Creating a Gem — Contributor Guide

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

This guide walks through creating a Viontin gem — a reusable plugin that extends the framework with assets, middleware, CLI commands, or runtime behavior.

---

## Gem Structure

A gem is a Rust crate under `products/gems/crates/`:

```
viontin-gem-example/
├── Cargo.toml
├── src/
│   └── lib.rs
├── resources/
│   └── views/
└── templates/
```

### Cargo.toml

```toml
[package]
name = "viontin-gem-example"
version = "0.1.0"
edition = "2024"
license = "MIT"
description = "[EXPERIMENTAL] Example Viontin gem"

[dependencies]
framework = { path = "../../../framework/crates/framework" }
gems = { path = "../gems" }
```

---

## Core Concepts

Every gem needs three things:

### 1. GemBuilder — Constructor Contract

```rust
pub trait GemBuilder: Sized {
    fn load() -> Self;
}
```

Every gem must implement `GemBuilder`. The `load()` method is the standard constructor — always parameterless. Configuration is done via builder method chaining.

### 2. GemFacade — Lifecycle Hooks

```rust
pub trait GemFacade: Debug + Send + Sync {
    fn meta(&self) -> &GemMeta;
    fn before_build(&self) -> Result<()> { Ok(()) }
    fn after_build(&self) -> Result<()> { Ok(()) }
}
```

### 3. GemMeta — Identity

```rust
pub const META: GemMeta = GemMeta::new("my-gem", "0.1.0", "Does something", GemKind::Integration);
```

| Field | Description |
|-------|-------------|
| `name` | Unique gem identifier (lowercase, kebab-case) |
| `version` | Semver version |
| `description` | Short description |
| `kind` | `Integration`, `Platform`, `Database`, `Auth`, `DevTool`, `Theme`, `Custom(&str)` |
| `homepage` | Optional URL |

---

## Step 1: Define the Gem Struct

```rust
// src/lib.rs
use viontin_gems::{GemBuilder, GemMeta, GemKind, GemFacade};
use viontin_framework::Result;

pub const META: GemMeta = GemMeta::new(
    "example",
    "0.1.0",
    "Example gem",
    GemKind::Integration,
);

#[derive(Debug)]
pub struct ExampleGem {
    pub config_path: String,
}

impl GemBuilder for ExampleGem {
    fn load() -> Self {
        ExampleGem { config_path: String::new() }
    }
}

impl ExampleGem {
    pub fn config(mut self, path: &str) -> Self {
        self.config_path = path.into();
        self
    }
}
```

---

## Step 2: Implement GemFacade

```rust
impl GemFacade for ExampleGem {
    fn meta(&self) -> &GemMeta {
        &META
    }

    fn before_build(&self) -> Result<()> {
        println!("  [example] before_build: config at {}", self.config_path);
        // Load configuration, generate assets, validate setup...
        Ok(())
    }

    fn after_build(&self) -> Result<()> {
        println!("  [example] after_build: cleanup");
        Ok(())
    }
}
```

---

## Step 3: Add GemBinding (Optional)

If your gem needs to register middleware, providers, CLI commands, or routes into the framework, implement `GemBinding`:

```rust
use viontin_gems::GemBinding;
use viontin_framework::middleware::Middleware;

impl GemBinding for ExampleGem {
    fn gem_middlewares(&self) -> Vec<Box<dyn Middleware + 'static>> {
        vec![Box::new(ExampleMiddleware::new())]
    }

    fn gem_providers(&self) -> Vec<Box<dyn ServiceProvider + 'static>> {
        vec![Box::new(ExampleProvider::new())]
    }

    fn gem_commands(&self) -> Vec<Box<dyn Command + 'static>> {
        vec![Box::new(ExampleCommand)]
    }

    fn gem_routes(&self) -> Option<fn(Router) -> Router> {
        Some(|router| router.static_files("/example-assets", "vendor/example/assets"))
    }
}
```

### When to Use Each Binding

| Method | When to Use |
|--------|-------------|
| `gem_middlewares()` | You need to intercept every request (auth, CORS, logging, Inertia) |
| `gem_providers()` | You need to register services into the DI container |
| `gem_commands()` | Your gem provides CLI tooling |
| `gem_routes()` | Your gem needs to serve static assets from a vendor directory |

If your gem doesn't need any bindings, just add an empty implementation:

```rust
impl GemBinding for ExampleGem {}
```

> Note: `GemBinding` requires `GemBuilder` — your gem already has `load()` from Step 1, so only the `GemBinding` block needs to be added.

---

## Step 4: Create a SimpleGem (No Struct Needed)

For simple hooks without a full struct:

```rust
use viontin_framework::gem::{SimpleGem, GemMeta, GemKind};

let gem = SimpleGem::new(
    GemMeta::new("hello", "0.1.0", "Says hello", GemKind::DevTool),
)
.on_before_build(Box::new(|| {
    println!("  [hello] Hello from gem!");
    Ok(())
}));
```

---

## Step 5: Create a DynamicGem (WASM-based)

For gems loaded from WebAssembly at runtime:

```rust
use viontin_framework::gem::{DynamicGem, GemMeta, GemKind};

let gem = DynamicGem::new(
    GemMeta::new("wasm-plugin", "0.1.0", "WASM plugin", GemKind::Custom("wasm")),
    "./gems/plugin.wasm",
);
```

---

## Full Example Gem

```rust
use std::path::Path;
use viontin_gems::{GemBuilder, GemMeta, GemKind, GemFacade, GemBinding};
use viontin_framework::Result;

pub const META: GemMeta = GemMeta::new(
    "asset-bundler",
    "0.1.0",
    "JS/CSS asset bundler",
    GemKind::DevTool,
);

#[derive(Debug)]
pub struct AssetBundler {
    entry: String,
    output_dir: String,
}

impl GemBuilder for AssetBundler {
    fn load() -> Self {
        AssetBundler {
            entry: String::new(),
            output_dir: String::new(),
        }
    }
}

impl AssetBundler {
    pub fn entry(mut self, path: &str) -> Self {
        self.entry = path.into();
        self
    }

    pub fn output(mut self, dir: &str) -> Self {
        self.output_dir = dir.into();
        self
    }
}

impl GemFacade for AssetBundler {
    fn meta(&self) -> &GemMeta { &META }

    fn before_build(&self) -> Result<()> {
        println!("  [bundler] Building {} → {}", self.entry, self.output_dir);
        // Invoke esbuild / webpack / vite here
        Ok(())
    }
}

impl GemBinding for AssetBundler {
    fn gem_routes(&self) -> Option<fn(viontin_framework::server::Router) -> viontin_framework::server::Router> {
        let dir = self.output_dir.clone();
        Some(move |r| r.static_files("/assets", &dir))
    }
}
```

### Usage

```rust
boot()
    .gem(AssetBundler::load()
        .entry("src/index.js")
        .output("public/assets/")
    )
    .serve(":3000");
```

---

## API Reference

### Traits

| Trait | Super-trait | Purpose |
|-------|-------------|---------|
| `GemBuilder` | `Sized` | Constructor contract — every gem must implement `fn load() -> Self` |
| `GemFacade` | `Debug + Send + Sync` | Lifecycle hooks — `meta()`, `before_build()`, `after_build()` |
| `GemBinding` | `GemFacade + GemBuilder` | Auto-wiring — middleware, providers, commands, routes |

### GemBuilder

| Method | Return | Description |
|--------|--------|-------------|
| `load()` | `Self` | Standard parameterless constructor |

### GemFacade

| Method | Return | Description |
|--------|--------|-------------|
| `meta()` | `&GemMeta` | Return gem identity metadata (REQUIRED) |
| `before_build()` | `Result<()>` | Pre-boot hook — asset generation, validation |
| `after_build()` | `Result<()>` | Post-boot hook — cleanup, logging |

### GemBinding

| Method | Return | Description |
|--------|--------|-------------|
| `gem_middlewares()` | `Vec<Box<dyn Middleware>>` | Register global middlewares |
| `gem_providers()` | `Vec<Box<dyn ServiceProvider>>` | Register service providers |
| `gem_commands()` | `Vec<Box<dyn Command>>` | Register CLI commands |
| `gem_routes()` | `Option<fn(Router) -> Router>` | Configure HTTP routes |

### GemMeta

| Field | Type | Description |
|-------|------|-------------|
| `name` | `&'static str` | Unique identifier (kebab-case) |
| `version` | `&'static str` | Semver version |
| `description` | `&'static str` | Short description |
| `kind` | `GemKind` | Category: `Integration`, `Platform`, `Database`, `Auth`, `DevTool`, `Theme`, `Custom(&str)` |
| `homepage` | `&'static str` | Optional URL |

### GemKind

| Variant | Description |
|---------|-------------|
| `Integration` | Third-party integration (Inertia, etc.) |
| `Platform` | Platform extension |
| `Database` | Database driver or tool |
| `Auth` | Authentication provider |
| `DevTool` | Development tool (TailwindCSS, bundler) |
| `Theme` | Theme or styling |
| `Custom(&str)` | User-defined category |

---

## Testing Your Gem

```rust
#[test]
fn gem_meta_is_correct() {
    assert_eq!(META.name, "example");
    assert_eq!(META.version, "0.1.0");
}

#[test]
fn gem_binding_works() {
    let gem = ExampleGem::load().config("config.toml");
    assert!(gem.gem_middlewares().is_empty());
    assert!(gem.gem_providers().is_empty());
}
```

---

## Publishing

1. Add your gem to the workspace in `products/gems/Cargo.toml`
2. Tag a release
3. Users add it as a path or git dependency:

```toml
[dependencies]
viontin-gem-example = { git = "https://github.com/viontin/gems" }
```

---

## Best Practices

1. **Use `GemBinding`** if your gem needs to wire into the framework — don't make users call separate `.middleware()` or `.provider()` methods
2. **Keep `before_build()` fast** — it blocks the server start
3. **Fail gracefully** — return `Ok(())` with a warning instead of hard errors for non-critical failures
4. **Prefix your CSS classes and JS globals** to avoid conflicts
5. **Include templates** in a `resources/` directory within your gem crate
6. **Use `GemKind` accurately** to help users discover your gem by category
