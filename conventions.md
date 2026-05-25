# Conventions

> Last updated: 2026-05-25 — SoC, Feature Flag rules, DX-first, Unsafe Code; middleware+mail+ws+server refactored

## Module Naming

### Plural First

Module directories use **plural** names by default:

| ✅ Correct | ❌ Incorrect |
|-----------|-------------|
| `entities/` | `entity/` |
| `models/` | `model/` |
| `repositories/` | `repository/` |
| `services/` | `service/` |
| `controllers/` | `controller/` |
| `modules/` | `module/` |
| `validators/` | `validator/` |

Exception: trait/type names remain singular (`Entity`, `Model`, `Repository`, `Service`).

### No Abbreviations

Module names are **never abbreviated**:

| ✅ Correct | ❌ Incorrect |
|-----------|-------------|
| `notification/` | `notif/` |
| `rate_limit/` | `rate/` |
| `localization/` | `lang/` |
| `pagination/` | `page/` |
| `filesystem/` | `fs/` |
| `service_contract/` | `contract/` |

### Hierarchical Organization

Modules are organized hierarchically, not flat:

```
db/
├── mod.rs          # Connection, Value, Row, DbConfig (re-export from core)
├── query_log.rs    # Query logging (was flat top-level)
└── ...

http/
├── mod.rs          # Request, Response, StatusCode, etc. (re-export from core)
├── client.rs       # HTTP client (was flat http_client.rs)
└── form_request.rs # FormRequest trait

server/
├── mod.rs          # Router, Server, Handler
└── async_server.rs # Async server (was flat server_async.rs)

services/
├── mod.rs          # Service trait, DefaultService
└── contracts/      # ServiceContract, ServiceRegistry (was flat contract/)

support/
├── mod.rs          # Hasher, Encrypter traits
├── hash.rs         # SimpleHasher, etc.
├── str.rs          # String utilities
├── url.rs          # URL utilities
└── path.rs         # Path utilities (was flat path.rs)
```

### Flat File → Directory

Every flat `.rs` file at the crate root should be moved into an appropriate parent directory:

| Flat File | Target |
|-----------|--------|
| `query_log.rs` | `db/query_log.rs` |
| `http_client.rs` | `http/client.rs` |
| `server_async.rs` | `server/async_server.rs` |
| `path.rs` | `support/path.rs` |

---

## Dependency Rules

### Layer Order

```
viontin-core     ← shared contracts, minimal deps (serde, thiserror, serde_json)
       ↑
viontin-framework ← implementations, depends on core
       ↑
viontin-gems      ← plugin system, depends on core + framework
       ↑
viontin-gem-*     ← individual gems, depend on gems (not framework directly)
```

### Gem Dependency Rule

`viontin-gem-*` crates **must only directly depend on** `viontin-gems`. They may depend on `viontin-framework` only when they genuinely need framework-specific types (e.g., `Middleware`, `WsServer`).

Error types (`InternalResult`, `InternalError`) are re-exported through `viontin-gems` so gems can use them without deeper deps.

---

## Error Handling

### Single Error Type

All Viontin crates use `InternalError` / `InternalResult` from `viontin-core`:

```rust
use viontin_core::{InternalError, InternalResult};

fn do_something() -> InternalResult<()> {
    InternalError::not_found("user")
}
```

### Avoid Stringly-Typed Errors

Prefer `InternalError` variants over `Result<_, String>`:
- `InternalError::not_found(msg)` instead of `Err("Not found: ...".into())`
- `InternalError::connection(msg)` instead of `Err("Connection: ...".into())`
- `InternalError::validation(errors)` instead of `Err("Validation: ...".into())`

---

## Unsafe Code

### Zero Unsafe by Default

Viontin targets **zero `unsafe`** in framework code. Unsafe blocks are only permitted when:

1. **FFI boundary** — calling into C/Win32 APIs that have no safe Rust equivalent (e.g., console handle detection)
2. **Performance-critical hot path** — proven via benchmarks with safe-alternative comparison (requires `// SAFETY:` justification)
3. **Inline assembly** — platform-specific CPU features (requires test coverage)

### Safety Invariants

Every `unsafe` block MUST have a `// SAFETY:` comment explaining:

```
// SAFETY: <invariant that makes this safe>
// 1. <precondition 1>
// 2. <precondition 2>
// ...
```

### Preferred Alternatives

| Unsafe Pattern | Safe Alternative |
|---------------|-----------------|
| `std::env::set_var` | Store env vars in `OnceLock<HashMap>` and use custom lookup |
| `GetStdHandle` / Win32 FFI | `std::io::IsTerminal` (Rust 1.70+) |
| Raw pointer dereference | `&T` / `&mut T` references or `Pin<Box<T>>` |
| `std::mem::transmute` | `From` / `Into` / `TryFrom` / `bytemuck` crate |

### Audit

All `unsafe` blocks are tracked in [known-issues](dev-tools/known-issues) under `C-015`. New unsafe code requires maintainer review.

---

### Facade Crate (`viontin`)

The facade crate re-exports everything needed by end users:

```rust
pub use viontin_framework::entity;
pub use viontin_framework::entity::Entity;
```

**No aliases** unless there is a genuine name conflict. Module and type names live in different Rust namespaces and don't conflict.

Exception: `pub use viontin_core as core` — alias to avoid confusion with `std::core`.

### Framework Crate (`viontin-framework`)

The framework crate re-exports core types from `viontin-core` for convenience:

```rust
pub use viontin_core::{InternalError, InternalResult, Entity, Connection, ...};
```

This allows framework users to import everything from a single crate.

---

## Separation of Concerns (SoC)

### One Responsibility Per Module

Every module does exactly one thing. If a file or module has multiple responsibilities, split it.

```rust
// ❌ Don't: one module does everything
src/
└── cache.rs          // Cache + File I/O + Serialization + TTL logic

// ✅ Do: one concern per module
src/
├── cache/
│   ├── mod.rs        // CacheDriver trait + Cache facade
│   ├── memory.rs     // MemoryCache
│   ├── file.rs       // FileCache
│   ├── null.rs       // NullCache
│   ├── redis.rs      // RedisCache (behind feature flag)
│   └── serialization.rs  // Cache-specific (de)serialization
```

### Module Size Limit

A module should fit on one screen (≈60 lines of logic). If a file exceeds this:
1. Extract sub-concerns into submodules (e.g., `mail/smtp.rs`, `mail/log.rs`)
2. Extract shared logic into the parent `mod.rs`
3. Extract separate domains into sibling modules

### When to Split

| Signal | Action |
|--------|--------|
| Module has >3 `impl` blocks for different types | Split each type into its own file |
| Module mixes I/O with business logic | Extract I/O into a driver submodule |
| Module has `#[cfg]` gated sections | Extract into separate files with `#[cfg]` per file |
| Module is >200 lines | Audit for split opportunities |

### Example: mail/ — One transport per file

```
mail/
├── mod.rs   → Mailer trait, Mail facade, re-exports
├── log.rs   → LogTransport
├── array.rs → ArrayTransport
└── smtp.rs  → SmtpTransport (behind feature flag)
```

Each file has exactly one `struct` + `impl Trait`. Parent `mod.rs` only declares submodules and re-exports. This pattern applies to all large modules: identify distinct concerns and split each into its own file.

---

## Feature Flags

1. **Core ecosystem = direct dep.** If a dependency is essential to the framework's value proposition (HTTP, ORM, password hashing, serialization, caching), add it as a **direct dependency** — no feature flag. Users should not need to discover a feature flag to use a core feature.

2. **Feature flags for alternatives.** Use feature flags only when offering an **alternative implementation** of an existing trait (e.g., `async` Tokio server vs sync server, `aes` AES-256-GCM vs dev-only `SimpleEncrypter`).

3. **Feature flags for heavy deps.** Use feature flags when a dependency pulls in a large dependency tree that most users won't need (e.g., `http-client` for `ureq`).

4. **Forward to the meta-crate.** Every feature flag on `viontin-framework` must be forwarded in `viontin/Cargo.toml` so users can enable it via `viontin = { features = ["..."] }`.

### Current Flags

| Flag | Enables | Default | Category |
|------|---------|---------|----------|
| `async` | Tokio-based async HTTP server | No | Alternative implementation |
| `domain` | DDD building blocks (Domain, AggregateRoot) | No | Alternative implementation |
| `orm` | Removed — ORM is now a core direct dependency | — | Core (always on) |
| `http-client` | ureq-based HTTP client | No | Heavy dep |
| `shutdown` | SIGTERM/SIGINT handling | Yes | Core (default) |
| `aes` | AES-256-GCM encryption | No | Alternative implementation |
| `smtp` | SMTP email transport (lettre) | No | Heavy dep |

### What Goes In Without a Flag

| Dependency | Reason |
|-----------|--------|
| `serde` + `serde_json` | JSON serialization is core to any API framework |
| `bcrypt` | Password hashing is non-negotiable for auth |
| `thiserror` | Error handling is fundamental |
| `glob` | File pattern matching for config/assets |

---

## CLI Commands

All CLI commands use `:` for sub-command hierarchies:

```
viontin make:controller      viontin make:model
viontin make:service         viontin make:repository
viontin make:migration       viontin make:domain
```

45 commands total, zero `cargo` dependency at runtime.

---

## Developer Experience First

### API Design Principles

Every public API must pass the **"muscle memory" test**: a developer familiar with the framework should be able to guess the API correctly on the first try.

| Principle | Example (✅) | Anti-pattern (❌) |
|-----------|-------------|-------------------|
| **Methods are verbs** | `cache.get()`, `session.set()`, `queue.push()` | `cache.retrieveValue()`, `session.putData()` |
| **Constructors are nouns** | `Cache::memory()`, `Cache::file(path)` | `Cache::new_memory_backend()` |
| **Builder chains read left-to-right** | `QueryBuilder::table(conn, "users").where_eq("active", true).get()` | Nested function calls |
| **Facades delegate, not implement** | `Cache::memory()` wraps `Box<dyn CacheDriver>` | `Cache::memory()` implements caching inline |
| **No get_/set_ prefix** | `user.name()`, `user.set_name("x")` | `user.get_name()`, `user.set_name("x")` |

### Builder Pattern

All builders follow the same conventions:

```rust
// Step 1: Constructor (bare noun or ::new())
let cache = Cache::memory();

// Step 2: Chain setters (verbs or with_ prefix)
boot()
    .provider(MyProvider)         // plain verb
    .withProviders(vec![...])     // with_ for collections
    .withoutProvider("config")    // without_ for removal

// Step 3: Terminal method (action verb)
    .serve("127.0.0.1:3000");    // run, serve, get, finalize
```

| Pattern | When | Example |
|---------|------|---------|
| `.verb(value)` | Single item | `.provider(P)`, `.command(C)`, `.gem(G)` |
| `.with_noun(values)` | Multiple items | `.withProviders(v)`, `.withCommands(v)` |
| `.without_noun(name)` | Removal | `.withoutProvider("log")`, `.withoutCommands()` |

### Method Signature Conventions

```rust
// 1. Consume self for builders (builder pattern)
pub fn provider(mut self, p: impl ServiceProvider) -> Self

// 2. &self for facades (shared state)
pub fn get(&self, key: &str) -> Option<Value>

// 3. &mut self for state modification
pub fn set(&mut self, key: &str, value: Value)

// 4. Functions, not methods, for global access
config("app.name")         // not Config::get("app.name")
env("DATABASE_URL")        // not Environment::get("DATABASE_URL")
log_info!("started")       // not Logger::log(Level::Info, "started")
```

### Facade Naming

| Trait | Facade | Constructor Examples |
|-------|--------|-------------------|
| `CacheDriver` | `Cache` | `Cache::memory()`, `Cache::file(p)`, `Cache::null()` |
| `SessionDriver` | `Session` | `Session::memory()`, `Session::file(p)` |
| `Driver` (storage) | `Storage` | `Storage::local(p)`, `Storage::memory()` |
| `Mailer` | `Mail` | `Mail::log()`, `Mail::array()` |
| `AuthGuard` | `Auth` | `Auth::basic()`, `Auth::session()` |

### Error Ergonomics

```rust
// ✅ Return InternalError directly
fn find_user(id: i64) -> InternalResult<User> {
    let user = db.query("SELECT * FROM users WHERE id = ?", vec![id.into()])?
        .first()
        .ok_or_else(|| InternalError::not_found("user"))?;
    Ok(user)
}

// ✅ Use ? operator for automatic InternalError conversion
fn create_user(name: &str) -> InternalResult<i64> {
    let json: Value = serde_json::from_str(name)?;  // auto From<serde_json::Error>
    Ok(db.execute("INSERT INTO users ...")?)
}

// ❌ Never use unwrap() in framework code
let val = some_result.unwrap();  // FORBIDDEN

// ❌ Never use expect() without justification
let val = some_result.expect("msg");  // FORBIDDEN in library code
```

### CLI Signature Convention

```
# Laravel-inspired, space-separated:
make:controller {name} {--force} {--resource} {--type=default}
  │              │         │          │            │
  command name   required   flag      flag        option with default
                 argument
```

### Content Embedding

All content embedding macros follow the same pattern — filename without extension:

```rust
include_html!("pages/index.html")    // compile-time embedded HTML
include_md!("docs/guide.md")         // compile-time embedded Markdown
include_js!("assets/app.js")         // compile-time embedded JavaScript
```

### Testing Convention

```rust
// Architecture tests read like sentences:
#[test]
fn controller_naming() {
    arch("UserController")
        .toBeController()
      .and("PostController")
        .toBeController()
        .assert();
}

// Unit tests follow describe/it:
describe!("UserService", {
    it("creates a valid user", {
        expect(service.create(user)).toBeOk();
    });
});
```

### Rule of Thumb

If a developer needs to read the documentation more than once for the same API, the API is not intuitive enough. Fix the API, not the documentation.
