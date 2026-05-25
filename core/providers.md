> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Service Providers

**Module:** `viontin_framework::app::provider`

Service providers organize application bootstrap logic — registering services into the DI container and initializing them after all registrations are complete.

---

## ServiceProvider Trait

```rust
pub trait ServiceProvider: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn register(&self, app: &mut Application) {}
    fn boot(&self, app: &Application) {}
}
```

| Method | Phase | Description |
|--------|-------|-------------|
| `name()` | — | Unique provider identifier |
| `register()` | 1st | Bind services into the container |
| `boot()` | 2nd | Initialize after all registrations |

---

## Two-Phase Boot

```
1. register() for ALL providers
2. boot() for ALL providers (same order)
```

This ensures every provider can register its services before any provider tries to resolve them.

```rust
struct MailProvider;

impl ServiceProvider for MailProvider {
    fn name(&self) -> &str { "mail" }

    fn register(&self, app: &mut Application) {
        app.container.singleton(Mail::new(LogTransport));
    }

    fn boot(&self, app: &Application) {
        if let Some(mail) = app.container.resolve::<Mail>() {
            println!("Mail driver: {}", mail.mailer().name());
        }
    }
}
```

---

## Built-in Providers

| Provider | Name | register() | boot() |
|----------|------|-----------|--------|
| `EnvProvider` | `"env"` | — | Load `.env` |
| `ConfigProvider` | `"config"` | — | Load `config/*.json` |
| `LogProvider` | `"log"` | — | Init global logger |
| `QueueProvider` | `"queue"` | `SyncQueue` singleton | — |
| `EventsProvider` | `"events"` | `EventDispatcher` singleton | — |

---

## Registration

```rust
boot()
    .withoutProvider("log")
    .provider(CustomLogProvider)
    // ...
```

### Replace a Built-in Provider

```rust
// Replace LogProvider with custom one
boot()
    .provider(CustomLogProvider) // same name "log" → replaces
```

### Remove a Built-in Provider

```rust
boot()
    .withoutProvider("config")    // disable config loading
    .withoutProvider("log")      // disable default logger
    .withoutDefaultProviders()   // remove all 5 built-in providers
```

### DomainServiceProvider

When the `domain` feature is enabled, you can use `DomainServiceProvider` to register domain definitions and listeners:

```rust
use viontin::domain::{DomainServiceProvider, DomainConfig, Domain};

boot()
    .provider(DomainServiceProvider::new()
        .domain(DomainConfig::new(Domain::new("billing"))
            .listener(InvoicePaidHandler)
        )
    )
```

---

## Complete Example

```rust
use viontin::app::*;
use viontin::prelude::*;

struct DatabaseProvider;

impl ServiceProvider for DatabaseProvider {
    fn name(&self) -> &str { "database" }

    fn register(&self, app: &mut Application) {
        app.container.singleton(Database::connect());
    }

    fn boot(&self, app: &Application) {
        if let Some(db) = app.container.resolve::<Database>() {
            db.run_migrations().ok();
        }
    }
}

fn main() {
    boot()
        .provider(DatabaseProvider)
        .run_with(|_ctx| {
            // All providers registered and booted
        });
}
```