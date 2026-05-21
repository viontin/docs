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
        .gem(MyGem::load())
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
boot().gem(MyGem::load());

// Group
boot().withGems(vec![
    Box::new(/* GemBinding implementor */),
]);
```

> Every gem uses `SomeGem::load()` as its standard constructor. Configuration is done via builder chaining: `SomeGem::load().option(value)`.

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

### serve — Start HTTP Server (Shortcut)

```rust
boot()
    .get("/", || Response::html("<h1>Hello</h1>"))
    .serve("127.0.0.1:3000");
```

`serve(addr)` is a shortcut for `entry(|ctx| ctx.serve(addr)).run()`.

### run — Finalize + Execute

```rust
boot()
    .entry(|ctx| {
        ctx.serve(":3000");
    })
    .run();
```

Or with the shorthand:

```rust
boot()
    .run_with(|ctx| {
        ctx.serve(":3000");
    });
```

Flow:
1. `gems.before_build_all()` — plugin hooks
2. `app.run()` — register + boot all providers
3. `kernel.run(&args)` — dispatch CLI command if args exist
4. Finalize router — merge registry, attach middleware, WS
5. Call `entry` callback with `BootContext`

### BootContext — Runtime Access

```rust
.entry(|ctx| {
    ctx.serve(":3000");       // HTTP server (blocking)
    // ctx.cli();             // CLI dispatch (manual)
    // let app = ctx.into_inner(); // library mode
})
```

| Method | Purpose |
|--------|---------|
| `serve(addr)` | Start HTTP server (blocking) |
| `cli()` | Dispatch CLI commands |
| `into_inner()` | Return the underlying `Application` |

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

## API Reference

### Boot — Builder Methods

| Method | Input | Description |
|--------|-------|-------------|
| `provider(P)` | `impl ServiceProvider` | Register a service provider |
| `withProviders(Vec<Box<dyn ServiceProvider>>)` | Vec | Register multiple providers |
| `withoutProvider(name)` | `&str` | Remove a provider by name |
| `withoutDefaultProviders()` | — | Remove all 5 built-in providers |
| `command(C)` | `impl Command` | Register a CLI command |
| `withCommands(Vec<Box<dyn Command>>)` | Vec | Register multiple commands |
| `withoutCommand(name)` | `&str` | Remove a command by name |
| `withoutCommands()` | — | Remove all commands |
| `gem(G)` | `impl GemBinding + GemBuilder` | Register a gem plugin |
| `withoutGem(name)` | `&str` | Remove a gem by name |
| `withoutGems()` | — | Remove all gems |
| `middleware(M)` | `impl Middleware` | Register global middleware |
| `withMiddlewares(Vec<Box<dyn Middleware>>)` | Vec | Register multiple middlewares |
| `withoutMiddlewares()` | — | Remove all middlewares |
| `get(path, handler)` | `&str, fn` | Register GET route |
| `post(path, handler)` | `&str, fn` | Register POST route |
| `any(path, handler)` | `&str, fn` | Register any-method route |
| `routes(fn)` | `fn(Router) -> Router` | Configure router in closure |
| `ws(path, handler)` | `&str, impl WebSocketHandler` | Register WebSocket route |
| `ws_with_config(path, config, handler)` | `&str, WebSocketConfig, impl WebSocketHandler` | WebSocket with config |

### Boot — Terminal Methods

| Method | Input | Description |
|--------|-------|-------------|
| `serve(addr)` | `&str` | Start HTTP server (shortcut for `entry(ctx.serve).run()`) |
| `entry(f)` | `FnOnce(BootContext)` | Define application entry point |
| `run()` | — | Finalize + execute (init, CLI check, call entry) |
| `run_with(f)` | `FnOnce(BootContext)` | Shortcut for `entry(f).run()` |

### BootContext

| Method | Input | Description |
|--------|-------|-------------|
| `serve(addr)` | `&str` | Start HTTP server (blocking) |
| `cli()` | — | Dispatch CLI commands (exits if command found) |
| `into_inner()` | — | Consume context, return `Application` |

### Application

| Method | Input | Description |
|--------|-------|-------------|
| `new()` | — | Create with 5 built-in providers |
| `with(provider)` | `impl ServiceProvider` | Add or replace provider |
| `with_boxed(provider)` | `Box<dyn ServiceProvider>` | Add boxed provider |
| `without(name)` | `&str` | Remove a provider |
| `run()` | — | Register all providers, then boot all |

### Container

| Method | Input | Description |
|--------|-------|-------------|
| `singleton(value)` | `T: Any` | Register a singleton |
| `resolve::<T>()` | — | Resolve a singleton `Option<&T>` |
| `has::<T>()` | — | Check if singleton exists |
| `remove::<T>()` | — | Remove a singleton |

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
