> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Application Boot

**Module:** `viontin_framework::app`  
**Facade:** `viontin::boot()`

The boot sequence initializes the DI container, registers built-in service providers, and prepares the application for handling requests.

---

## Boot Function

```rust
use viontin::boot;

fn main() {
    boot()
        .provider(MyProvider)
        .command(MyCommand)
        .gem(MyGem)
        .get("/", handler)
        .serve("127.0.0.1:3000");
}
```

The `boot()` function creates a `Boot` builder holding:

| Component | Default | Description |
|-----------|---------|-------------|
| `Application` | 5 built-in providers | DI container + service providers |
| `Kernel` | Empty | CLI command registry |
| `Router` | Empty | HTTP route table |
| `WsRouter` | Empty | WebSocket route table |
| `GemRegistry` | Empty | Plugin registry |

---

## Boot Builder

### Provider Registration

```rust
boot()
    .provider(MyProvider)        // add or replace a provider
    .run(|_| {});
```

### CLI Commands

```rust
boot()
    .command(DeployCommand)
    .run(|_| {});
```

### HTTP Routes

```rust
boot()
    .get("/", home)
    .post("/submit", submit)
    .any("/fallback", catch_all)
    .routes(|r| r.get("/admin", admin))
```

### WebSocket Routes

```rust
boot()
    .ws("/chat", ChatHandler)
    .ws_with_config("/secure", config, SecureHandler)
```

### Gems

```rust
boot()
    .gem(TailwindGem)
```

---

## Terminal Methods

### serve — Start HTTP Server

```rust
boot()
    .get("/", || Response::html("<h1>Hello</h1>"))
    .serve("127.0.0.1:3000");
```

Flow:
1. `gems.before_build_all()` — plugin hooks
2. `app.run()` — register + boot all providers
3. `kernel.run(&args)` — dispatch CLI command if args exist
4. `router.extend_from_registry()` — merge route registry
5. `ws_router.attach(router)` — merge WebSocket routes
6. TCP server starts

### run — No Server Mode

```rust
boot()
    .command(BackupCommand)
    .run(|_| {
        println!("Running in library mode");
    });
```

Flow:
1. `gems.before_build_all()`
2. `app.run()`
3. `kernel.run(&args)` if CLI args exist
4. Executes the closure

---

## Application

```rust
pub struct Application {
    pub container: Container,
    providers: Vec<Box<dyn ServiceProvider>>,
}
```

### Methods

```rust
Application::new();           // create with 5 built-in providers
app.with(provider);           // add or replace provider by name
app.without("provider_name"); // remove a provider
app.run();                    // register all, then boot all
```

### Built-in Providers

| Provider | Name | Role |
|----------|------|------|
| `EnvProvider` | `"env"` | Load `.env` file if exists |
| `ConfigProvider` | `"config"` | Load `config/*.json` if directory exists |
| `LogProvider` | `"log"` | Initialize global logger with `default_logger()` |
| `QueueProvider` | `"queue"` | Register `SyncQueue` as singleton in container |
| `EventsProvider` | `"events"` | Register `EventDispatcher` as singleton |

---

## Container

Type-safe DI container keyed by `TypeId`:

```rust
let mut app = Application::new();

// Register a singleton
app.container.singleton(MyService::new());

// Resolve at runtime
let service = app.container.resolve::<MyService>();
```

```rust
pub struct Container {
    bindings: HashMap<TypeId, Box<dyn Any>>,
}

impl Container {
    pub fn singleton<T: Any>(&mut self, value: T);
    pub fn resolve<T: Any>(&self) -> Option<&T>;
    pub fn has<T: Any>(&self) -> bool;
    pub fn remove<T: Any>(&mut self);
}
```

---

## Complete Example

```rust
use viontin::{boot, html};

fn main() {
    boot()
        .provider(DatabaseProvider)
        .command(SeedCommand)
        .get("/", |_| Response::html(html!("pages/home.html")))
        .get("/users/:id", show_user)
        .ws("/chat", ChatHandler)
        .serve("127.0.0.1:3000");
}
```
