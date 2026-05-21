# Creating a Gem — Contributor Guide

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

This guide walks through creating a Viontin gem — a reusable plugin that extends the framework with assets, middleware, CLI commands, or runtime behavior.

---

## Gem Structure

A gem is a Rust crate under `repos/gems/crates/`:

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
viontin-framework = { path = "../../../framework/crates/framework" }
viontin-gems = { path = "../viontin-gems" }
```

---

## Core Concepts

Every gem needs two things:

### 1. GemFacade — Lifecycle Hooks

```rust
pub trait GemFacade: Debug + Send + Sync {
    fn meta(&self) -> &GemMeta;
    fn before_build(&self) -> Result<()> { Ok(()) }
    fn after_build(&self) -> Result<()> { Ok(()) }
}
```

### 2. GemMeta — Identity

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
use viontin_framework::Result;
use viontin_framework::gem::{GemMeta, GemKind, GemFacade};

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

impl ExampleGem {
    pub fn new(config_path: &str) -> Self {
        ExampleGem { config_path: config_path.into() }
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
use viontin_framework::gem::GemBinding;
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
use viontin_framework::Result;
use viontin_framework::gem::{GemMeta, GemKind, GemFacade, GemBinding};

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

impl AssetBundler {
    pub fn new(entry: &str, output_dir: &str) -> Self {
        AssetBundler {
            entry: entry.into(),
            output_dir: output_dir.into(),
        }
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
    let gem = ExampleGem::new("config.toml");
    assert!(gem.gem_middlewares().is_empty());
    assert!(gem.gem_providers().is_empty());
}
```

---

## Publishing

1. Add your gem to the workspace in `repos/gems/Cargo.toml`
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
