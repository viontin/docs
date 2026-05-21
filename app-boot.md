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
| `MiddlewareChain` | Empty | Global middleware chain |

---

## Boot Builder

### Provider Registration

```rust
// Single
boot().provider(MyProvider);

// Group
boot().withProviders(vec![
    Box::new(DatabaseProvider),
    Box::new(CacheProvider),
]);
```

### CLI Commands

```rust
// Single
boot().command(DeployCommand);

// Group
boot().withCommands(vec![
    Box::new(BackupCmd),
    Box::new(RestoreCmd),
]);
```

### Gems

```rust
// Single
boot().gem(MyGem);

// Group
boot().withGems(vec![
    Box::new(viontin_gem_inertia::Inertia::new("views/app.html")),
]);
```

### Middleware

```rust
// Single
boot().middleware(LoggerMw);

// Group
boot().withMiddlewares(vec![
    Box::new(LoggerMw),
    Box::new(CorsMw),
]);
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

### Removing Defaults

```rust
boot()
    .withoutProvider("config")    // disable config loading
    .withoutProvider("log")      // disable default logger
    .withoutDefaultProviders()   // remove all 5 built-in providers
    .withoutCommand("greet")     // remove a specific command
    .withoutCommands()           // clear all commands
    .withoutGem("tailwind")      // remove a specific gem
    .withoutGems()               // clear all gems
    .withoutMiddlewares()        // clear all middlewares
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
5. Global middlewares applied to router
6. `ws_router.attach(router)` — merge WebSocket routes
7. TCP server starts

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
app.with_boxed(provider);     // add boxed provider
app.without("provider_name"); // remove a provider
app.without_boxed(provider);   // remove boxed provider
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
        .withMiddlewares(vec![
            Box::new(LoggerMiddleware),
        ])
        .get("/", |_| Response::html(html!("pages/home.html")))
        .get("/users/:id", show_user)
        .ws("/chat", ChatHandler)
        .serve("127.0.0.1:3000");
}
```
