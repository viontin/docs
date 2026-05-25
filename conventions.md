# Conventions

> Last updated: 2026-05-25

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

## Re-exports

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

## Feature Flags

| Flag | Enables | Default |
|------|---------|---------|
| `async` | Tokio-based async HTTP server | No |
| `domain` | DDD building blocks (Domain, AggregateRoot) | No |
| `orm` | ORM integration | No (default in meta-crate) |
| `http-client` | ureq-based HTTP client | No |
| `shutdown` | SIGTERM/SIGINT handling | Yes (in meta-crate) |
| `aes` | AES-256-GCM encryption | No |

---

## CLI Commands

All CLI commands use `:` for sub-command hierarchies:

```
viontin make:controller      viontin make:model
viontin make:service         viontin make:repository
viontin make:migration       viontin make:domain
```

45 commands total, zero `cargo` dependency at runtime.
