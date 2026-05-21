# Gems (Plugin System)

**Module:** `viontin_framework::gem`

A plugin system that extends Viontin at build time and runtime. Gems hook into the application lifecycle — before and after build.

---

## GemFacade Trait

```rust
pub trait GemFacade: Debug + Send + Sync {
    fn meta(&self) -> &GemMeta;
    fn before_build(&self) -> Result<()> { Ok(()) }
    fn after_build(&self) -> Result<()> { Ok(()) }
}
```

| Method | Phase | Description |
|--------|-------|-------------|
| `meta()` | — | Gem metadata (name, version, kind) |
| `before_build()` | Build | Hook before build starts |
| `after_build()` | Build | Hook after build completes |

---

## GemMeta

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
const META: GemMeta = GemMeta::new("my-gem", "0.1.0", "Does things", GemKind::Integration)
    .homepage("https://example.com");
```

---

## GemKind

```rust
pub enum GemKind {
    Integration, Platform, Database, Auth, DevTool, Theme,
    Custom(&'static str),
}
```

---

## SimpleGem

For gems with simple hooks:

```rust
use viontin::prelude::*;

let gem = SimpleGem::new(GemMeta::new("my-gem", "1.0.0", "Description", GemKind::Integration))
    .on_before_build(Box::new(|| {
        println!("  [gem] before build hook");
        Ok(())
    }))
    .on_after_build(Box::new(|| {
        println!("  [gem] after build hook");
        Ok(())
    }));
```

---

## DynamicGem

For WASM-based gems loaded at runtime:

```rust
let gem = DynamicGem::new(
    GemMeta::new("wasm-gem", "0.1.0", "WASM plugin", GemKind::Custom("wasm")),
    "./gems/wasm-gem.wasm",
);
```

---

## GemRegistry

```rust
pub struct GemRegistry {
    installed: HashMap<String, Box<dyn GemFacade>>,
}
```

### Methods

```rust
registry.register(simple_gem);
registry.register_simple(simple_gem);
registry.register_dynamic(dynamic_gem);
registry.get("gem-name");              // Option<&dyn GemFacade>
registry.all();                         // Vec<&dyn GemFacade>
registry.by_kind(GemKind::Database);   // Vec<&dyn GemFacade>
registry.before_build_all()?;          // run all before_build hooks
registry.after_build_all()?;           // run all after_build hooks
```

---

## Integration

```rust
use viontin::boot;

fn main() {
    boot()
        .gem(SimpleGem::new(
            GemMeta::new("tailwind", "1.0.0", "TailwindCSS", GemKind::DevTool),
        ))
        .run(|_| {});
}
```

Gems execute their hooks during the boot lifecycle:

```
boot()
  ├── gem(MyGem)              → register
  ├── .serve(":3000")
  │     ├── gems.before_build_all()  ← hooks run here
  │     ├── app.run()
  │     └── server starts
  └── gems.after_build_all()         ← hooks run here
```
