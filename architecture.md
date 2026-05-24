# Viontin Architecture

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

**Version:** 0.1.0  
**Tagline:** Zero to One, Scale-up Easily

### GemBinding — Standard Plug

Gems declare what they wire into the framework via `GemBinding`:

```rust
impl GemBinding for Inertia {
    fn gem_middlewares(&self) -> Vec<Box<dyn Middleware>> {
        vec![Box::new(InertiaMiddleware::new())]
    }
}
```

`Boot::gem()` auto-detects `GemBinding` and registers middleware, providers, commands, and routes automatically.

### Boot Builder & without\*() Methods

The `Boot` builder provides `without*()` methods to disable default behaviors:

```rust
boot()
    .withoutProvider("config")    // disable config loading
    .withoutDefaultProviders()   // remove 5 built-in providers
    .withoutCommands()           // clear CLI commands
    .withoutGems()               // clear plugins
    .withoutMiddlewares()        // clear middlewares
```

### Optional Features

| Feature | Flag | Description |
|---------|------|-------------|
| Async server | `async` | Tokio-based async HTTP server |
| Domain-Driven Design | `domain` | DDD building blocks (Domain, AggregateRoot, Repository) |
| Viontin ORM | `orm` | Enable `viontin-orm` integration via `viontin_framework::orm` |

---

## 1. Platform Overview

Viontin is a full-stack Rust application platform for building web services, CLI tools, terminal (TUI) applications, batch processing systems, and mixed-mode binaries — all within a single cohesive framework.

The platform is organized as a monorepo with four independent crate families:

```
viontin/
├── repos/
│   ├── framework/         # Core platform (3 crates + meta-crate)
│   ├── gems/              # Plugin system (WASM-based)
│   └── orm/               # Multi-driver ORM
├── examples/
├── docs/
└── scripts/
```

---

## 2. Crate Architecture

### 2.1 Dependency Graph

```
                    ┌─────────────────────────────────────────────┐
                    │               viontin (meta-crate)           │
                    │     re-exports: fw, tui, gems, orm           │
                    │     provides: boot(), domain!(), html!()     │
                    └─────────────────────────────────────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           ▼                       ▼                       ▼
┌──────────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
│  viontin-framework    │ │  viontin-tui      │ │  viontin-cli         │
│  core library         │ │  terminal UI      │ │  binary (33 cmds)   │
│  35 public modules    │ │  prompts/styling  │ │                      │
└──────────────────────┘ └──────────────────┘ └──────────────────────┘
           │
           │ (optional)
           ├───────────────────────┐
           │                       │
           ▼                       ▼
┌──────────────────────┐ ┌──────────────────────┐
│  viontin-gems         │ │  viontin-orm         │
│  plugin registry      │ │  ORM core + drivers  │
│  WASM loader          │ │  pg/mysql/sqlite     │
└──────────────────────┘ └──────────────────────┘
```

### 2.2 Crate Responsibilities

| Crate | Type | Dependencies | Purpose |
|-------|------|-------------|---------|
| `viontin` | Meta-crate | `framework`, `tui`, `gems`, `glob` | Unified facade: re-exports for end users, boot lifecycle, macros |
| `viontin-framework` | Library | `serde`, `serde_json`, `thiserror`, `glob` | Core platform: all types, traits, implementations, runtime |
| `viontin-tui` | Library | `framework`, `crossterm` (opt), `terminal_size`, `unicode-width` | Terminal toolkit: ANSI styling, interactive prompts, signature validator |
| `viontin-cli` | Binary | `tui`, `framework`, `notify`, `regex` | CLI executable: 42 commands, project scanner |
| `viontin-gems` | Library | `framework`, `linkme`, `serde` (opt) | Plugin system: GemRegistry, WASM lifecycle |
| `viontin-orm` | Library (optional) | (none — standalone) | ORM core: QueryBuilder, Schema, Migration, DatabaseType, NoSqlConnection |
| `viontin-orm-pg` | Library (optional) | `viontin-orm` | PostgreSQL driver |
| `viontin-orm-mysql` | Library (optional) | `viontin-orm` | MySQL driver |
| `viontin-orm-sqlite` | Library (optional) | `viontin-orm` | SQLite driver |

---

## 3. Three-Layer Platform Model

The `viontin-framework` crate follows a strict three-layer design:

```
┌──────────────────────────────────────────────────────────────────┐
│                      RUNTIME LAYER                                │
│  Executable infrastructure: server, router, drivers, scheduler   │
│  Depends on: Core + Types                                         │
│                                                                   │
│  server  routing  caching  sessions  storage  mail               │
│  notif   db-query csrf    rate-limit schedule  i18n              │
│  debug   errors   ws      arch-test  domain  fs                  │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│                      CORE LAYER                                   │
│  Concrete implementations of traits                               │
│  Depends on: Types                                                 │
│                                                                   │
│  Application  ConfigRepository  Logger  Authentication           │
│  EventDispatcher  SyncQueue  ValidatorGroup  SimpleEncrypter     │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│                      TYPES & TRAITS LAYER                         │
│  Pure interfaces + value types — zero runtime dependencies       │
│                                                                   │
│  http  auth  cache  collection  config  database  env           │
│  events  log  mail  notif  page  queue  rate  route             │
│  schedule  semver  session  storage  support  validator         │
└──────────────────────────────────────────────────────────────────┘
```

### 3.1 Types & Traits Layer

Pure contract definitions with no runtime dependencies. These modules **only** define traits and value types:

| Module | Key Contracts |
|--------|--------------|
| `auth` | `AuthGuard`, `AuthUser`, `AuthResult` |
| `cache` | `CacheDriver` (get/set/has/forget/clear) |
| `collection` | `Collection<T>` (map/filter/reduce/chunk/unique/sum/avg) |
| `config` | `ConfigValue` enum (String/Int/Float/Bool/Array/Object/Null) |
| `database` | `Connection`, `ConnectionPool`, `Value`, `Row`, `DbConfig` |
| `env` | `Environment` enum (Local/Dev/Staging/Prod/Testing) |
| `events` | `Event`, `Listener`, `Subscriber`, `Subscribable` |
| `http` | `Request`, `Response`, `StatusCode`, `Method`, `Headers`, `Uri`, `Cookie` |
| `log` | `Level`, `LogEntry`, `LogChannel` |
| `mail` | `Mailer`, `Envelope`, `Attachment` |
| `notif` | `Notification`, `Notifiable`, `Channel` |
| `page` | `Page<T>`, `PaginationLinks` |
| `queue` | `Job`, `Driver` |
| `rate` | `RateLimiter` |
| `route` | `RouteDefinition`, `RouteMethod` |
| `schedule` | `ScheduledJob`, `cron_matches()` |
| `semver` | `Version`, `VersionReq`, `Meta`, `Compatibility` |
| `session` | `SessionDriver` |
| `storage` | `Driver` |
| `support` | `Hasher`, `Encrypter`, `SimpleHasher`, string/URL utilities |
| `testing` | `ArchRule`, `ArchResult`, `ArchSeverity`, `ArchFinding` |
| `validator` | `Validator`, `Outcome`, `Finding`, `Severity`, `Context` |

Each trait is kept minimal — single-responsibility with 2–6 methods.

### 3.2 Core Layer

Concrete implementations of the type-layer traits, plus application infrastructure:

| Module | Key Implementations |
|--------|-------------------|
| `app` | `Application`, `Container` (TypeId-based DI), `ServiceProvider`, built-in providers (Env/Config/Log/Queue/Events) |
| `config` | `ConfigRepository` (JSON + env overlay), `ConfigLoader`, global `config()`/`config_set()` |
| `logging` | `Logger`, `StdoutLog`, global `log_info()`/`log_error()`/`log_debug()`/`log_warning()` |
| `auth` | `BasicGuard`, `SessionGuard`, `Auth` facade |
| `events` | `EventDispatcher` (pub/sub with subscriber registration) |
| `queue` | `SyncQueue`, `Queue` facade |
| `validator` | `ValidatorGroup` (composite validation) |
| `encryption` | `SimpleEncrypter` (XOR-based — dev only, NOT secure for production) |

### 3.3 Runtime Layer

Executable infrastructure — servers, drivers, and runtime utilities:

| Module | Key Implementations |
|--------|-------------------|
| `server` | `Server` (TCP), `Router`, `Handler` type alias, per-connection thread pool |
| `routing` | `RouteRegistry`, global `register()`/`register_handler()`/`finalize()`, `RouteProvider` |
| `caching` | `MemoryCache`, `FileCache`, `NullCache`, `Cache` facade |
| `sessions` | `MemorySession`, `FileSession`, `Session` facade |
| `storage` | `LocalStorage`, `MemoryStorage`, `Storage` facade |
| `mail` | `LogTransport`, `ArrayTransport`, `Mail` facade |
| `notif` | `MailChannel`, `DatabaseChannel`, `Notif` facade |
| `db` | `QueryBuilder` (fluent SQL: select/where/join/orderBy/groupBy/limit/offset) |
| `fs` | `read()`, `write()`, `copy()`, `delete()`, `ensure_dir()`, `TempDir`, `FileInfo`, `find_files()` |
| `csrf` | `CsrfManager`, `CsrfConfig` (token generation/validation, session-backed) |
| `rate` | `TokenBucketLimiter` (backed by Cache driver) |
| `schedule` | `Scheduler` (cron expression engine, job runner) |
| `lang` | `JsonTranslator` (JSON-based with locale fallback), `trans()`/`choice()`/`locale()` |
| `debug` | `dump()`, `dd()`, `Profiler`, `benchmark()`, `memory_usage()` |
| `error`/`errors` | `FrameworkError` enum, `HttpError`, `ErrorReport`, error page rendering |
| `domain` | `Domain`, `DomainBoundary`, `DomainViolation`, `register()`/`check_all()` |
| `testing` | `ArchChecker`, built-in `ArchRule` impls: `IsPascalCase`, `IsSnakeCase`, `DoesNotDependOn`, etc. |
| `gem` | `GemFacade`, `GemRegistry`, `GemMeta`, `GemKind`, `SimpleGem`, `DynamicGem` |
| `ws` | `WebSocketHandler`, `WsRouter`, `WsServer`, frame encode/decode, handshake |
| `cli` | `Command`, `Kernel`, `Input`, `Output`, `ExitCode`, `Signature`, `ArgDef`, `OptDef`, `Spinner`, `ProgressBar` |

---

## 4. Module Dependency Map

Within `viontin-framework`, modules depend on each other as follows:

```
app → config, env, log, queue, events
auth → support::hash (SimpleHasher, Hasher)
csrf → session
encryption → support (Encrypter trait)
notif → mail (Envelope)
rate → cache (CacheDriver)
route → http (Method), server (Handler)
route::provider → app (ServiceProvider), route
server → http (Request, Response, etc.)
ws → http, server (Router)
cli::output → log (LogChannel)
cli::input → cli::command (Signature)
cli::kernel → cli::command, cli::input, cli::exit, cli::output
gem::validator → validator, semver
semver::compat → semver::version, semver::constraint
semver::constraint → semver::version

All standalone modules (no internal deps):
collection  fs  page  path  support  cli (partial)
```

**Standalone modules** — no internal framework dependencies:
- `collection` — pure data structure
- `env` — file parsing only
- `fs` — OS-level file operations
- `page` — pure pagination math
- `path` — path resolution helpers
- `support` — utilities (hashing, string, URL)
- `debug` — diagnostic helpers
- `testing::arch` — architecture rule engine

---

## 5. Boot Lifecycle

The `viontin` meta-crate provides the unified boot sequence via `boot()`:

```
boot()
  │
  ├── Application::new()
  │     └── Container::new()
  │     └── Built-in providers:
  │           EnvProvider     → loads .env
  │           ConfigProvider  → loads config/*.json
  │           LogProvider     → initializes Logger
  │           QueueProvider   → initializes SyncQueue
  │           EventsProvider  → initializes EventDispatcher
  │
  ├── Kernel::new()
  │     └── Empty command registry (filled by user)
  │
  ├── Router::new()
  │     └── Empty route table (filled by user or .routes())
  │
  ├── WsRouter::new()
  │     └── Empty WebSocket route table
  │
  └── GemRegistry::new()
        └── Empty gem registry (filled by user)
```

### 5.1 Builder Methods

```rust
Boot
  ├── .provider(P)          → register a ServiceProvider
  ├── .command(C)           → register a CLI Command
  ├── .gem(G)               → register a Gem plugin (requires `GemBuilder` + `GemBinding`)
  ├── .routes(fn)           → configure Router in closure
  ├── .get("/", handler)    → register GET route
  ├── .post("/", handler)   → register POST route
  ├── .any("/", handler)    → register route for any method
  ├── .ws("/", handler)     → register WebSocket handler
  ├── .ws_with_config(...)  → register WebSocket with config
  │
   ├── .entry(f)             → define application entry point
   ├── .serve("addr")        → HTTP server (shortcut for `entry(\|ctx\| ctx.serve(addr)).run()`)
   ├── .run()                → finalize + execute
   └── .run_with(f)          → shortcut for `entry(f).run()`
```

### 5.2 Serve Flow

```
.serve("127.0.0.1:8080")
  │
  ├── gems.before_build_all()    → plugin pre-build hooks
  │
  ├── app.run()
  │     └── For each provider:
  │           1. register() → bind singletons into Container
  │           2. boot()     → post-registration setup
  │     (Order: Env → Config → Log → Queue → Events)
  │
  ├── If CLI args exist:
  │     └── kernel.run(&args) → dispatch to registered Command
  │
  ├── router.extend_from_registry() → merge RouteRegistry into Router
  │
  ├── ws_router.attach(router)     → merge WebSocket routes
  │
  └── server.run("addr")           → start TCP listener
```

### 5.3 Run Flow

```
.run()
  │
  ├── gems.before_build_all()
  ├── app.run()
  ├── If CLI args exist → kernel.run() → exit
  └── entry(|ctx| { ... })  ← user callback
```

This mode is used for CLI-only tools, library-style usage, or custom application loops where no HTTP server is needed.

---

## 6. Architectural Patterns

### 6.1 Facade Pattern

The framework uses a consistent facade pattern for swappable backends:

```
┌──────────────┐     ┌───────────────────┐
│   Facade     │────▶│  Box<dyn Trait>    │
│  (user API)  │     │  (inner driver)    │
└──────────────┘     └───────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Impl A   │ │ Impl B   │ │ Impl C   │
        └──────────┘ └──────────┘ └──────────┘
```

Each facade:
1. Wraps `Box<dyn Trait>` 
2. Delegates all calls to the inner driver
3. Adds cross-cutting concerns (key prefixing, auto-flush, default routing)

| Facade | Trait | Built-in Drivers |
|--------|-------|-----------------|
| `Cache` | `CacheDriver` | `MemoryCache`, `FileCache`, `NullCache` |
| `Session` | `SessionDriver` | `MemorySession`, `FileSession` |
| `Storage` | `Driver` | `LocalStorage`, `MemoryStorage` |
| `Mail` | `Mailer` | `LogTransport`, `ArrayTransport` |
| `Queue` | `Driver` | `SyncQueue` |
| `Notif` | — | `MailChannel`, `DatabaseChannel` |
| `Auth` | `AuthGuard` | `BasicGuard`, `SessionGuard` |

The pattern allows users to swap implementations at configuration time without changing application code.

### 6.2 Service Provider Pattern

Inspired by Laravel's service container, the DI system works in two phases:

```
Application::new()
  │
  ├── .with(MyProvider)       → register provider
  ├── .without::<ProviderT>()  → remove built-in provider
  │
  └── .run()
        ├── Phase 1: register()
        │     └── For each provider:
        │           provider.register(&mut container)
        │           → binds singletons: container.singleton::<T>(instance)
        │
        └── Phase 2: boot()
              └── For each provider (same order):
                    provider.boot(&container)
                    → reads resolved dependencies
```

**Built-in providers** (registered in order):

1. `EnvProvider` — loads `.env` into process environment
2. `ConfigProvider` — reads `config.json` + `config.{env}.json`, builds `ConfigRepository`
3. `LogProvider` — initializes `Logger` with `StdoutLog` channel
4. `QueueProvider` — initializes `SyncQueue` driver
5. `EventsProvider` — initializes `EventDispatcher`

**Container implementation:**

```rust
struct Container {
    bindings: HashMap<TypeId, Box<dyn Any>>,
}
```

Type-safe, keyed by `TypeId` — no string-based lookup, no downcasting errors at runtime.

### 6.3 Domain-Driven Design Enforcement

Viontin enforces bounded contexts at the application level:

```
Domain {
    name: &'static str,
    allows: &'static [&'static str],     // allowed dependencies
    provides: &'static [&'static str],   // public API surface
}
```

**Registration (via `domain!` macro):**

```rust
domain!(billing, allows: [order, payment]);
// expands to:
//   Domain { name: "billing", allows: &["order", "payment"], provides: &[] }
//   register_domain(d);
```

**Enforcement flow:**

```
viontin check --arch
  │
  └── check_domains()
        ├── For each domain in global registry:
        │     └── DomainBoundary::scan_imports(path)
        │           └── Regex-based `use` statement analysis
        │                 ├── Found import NOT in allows → Error (✘)
        │                 └── Found import IN allows but not provides → Warning (⚠)
        │
        └── Return Vec<DomainViolation>
```

The `allows` list controls which domains may be imported. The `provides` list designates the public API surface — imports from `allows`-listed domains that bypass `provides` generate warnings.

**Levels of enforcement:**

| Level | Behavior |
|-------|----------|
| 0–1 | No domains, no checking |
| 2 | Domains detected, `viontin check --arch` enforces boundaries |
| 3+ | Boundaries become service contracts (CI-enforced) |

---

## 7. CLI Command System

### 7.1 Architecture

The CLI system spans two crates:

```
viontin-tui                    viontin-cli
┌─────────────────────┐       ┌──────────────────────┐
│  Re-exports:         │       │  Binary: main.rs      │
│  Command (trait)     │       │  Commands:            │
│  Kernel (dispatch)   │       │    new.rs  build.rs   │
│  Input/Output         │       │    dev.rs  check.rs   │
│  ExitCode             │       │    make.rs inspect.rs │
│                       │       │    add.rs  run.rs     │
│  Adds:                │       │    pkg.rs  test.rs    │
│    styling (ANSI)     │       │                      │
│    prompts (crossterm)│       │  project.rs (scanner) │
│    validator          │       └──────────────────────┘
└─────────────────────┘
```

### 7.2 Command Trait

```rust
trait Command {
    fn signature(&self) -> &Signature;    // argument/option schema
    fn description(&self) -> &str;
    fn handle(&self, input: Input, output: Output) -> ExitCode;
}
```

**Signature syntax** (Laravel-inspired):

```
"make:controller {name} {--force} {--resource} {--type=default}"
  │              │         │          │            │
  command name   required   flag      flag        option
                 argument                      with default
```

### 7.3 42 Commands

```
Level 0 — Core (project lifecycle)
  new      build    dev     run     check   test    add
  init

Cargo Management
  clean    doc      fix     bench   tree    package  metadata

Publishing & Registry
  publish  update   install uninstall  search

Code Quality
  fmt      clippy

Level 1 — Scaffolding (make:*)
  make:controller   make:middleware   make:model    make:route
  make:command      make:event        make:job      make:mail
  make:notification make:query        make:module   make:service
  make:repository

Level 2 — Domain-Driven Design (DDD)
  make:domain         (bounded contexts)
  make:aggregate      (aggregate roots, event sourcing)
  make:entity         (domain entities)
  make:value-object   (immutable value objects)

Level 3 — Microservices
  make:service-contract

Level 4 — MVC Templates
  make:view

Level 5 — Inspection
  inspect (--domains, --models, --routes, --commands)
```

### 7.4 Dispatch Flow

```
kernel.run(&args)
  │
  ├── Parse args to extract command name
  ├── Lookup Command by signature name match
  ├── If found:
  │     ├── signature.parse(args) → Input (validated)
  │     ├── command.handle(input, output) → ExitCode
  │     └── return ExitCode
  │
  └── If not found:
        ├── Show "Command not found"
        └── Suggest similar commands
```

---

## 8. TUI Toolkit

The `viontin-tui` crate layers interactive terminal capabilities on top of the framework's CLI abstractions:

| Component | Description | Backend |
|-----------|-------------|---------|
| `styling` | ANSI style helpers: bold, dim, italic, underline, foreground/background colors (8-color, 256-color, truecolor) | Pure ANSI escape codes |
| `prompts` | Interactive: `text()`, `select()`, `confirm()`, `password()` | `crossterm` (optional, behind `prompts` feature) |
| `validator` | CLI signature parsing, argument/option validation | Standalone (reimplements Laravel-style signature grammar) |

The TUI crate re-exports `Kernel`, `Command`, `Input`, `Output`, and `ExitCode` from `viontin-framework::cli` — making it the primary entry point for CLI application authors.

---

## 9. HTTP Server & Routing

### 9.1 Server Architecture

```
Server
  │
  ├── TcpListener::bind("addr")
  │
  └── accept loop (per-connection thread)
        │
        ├── parse HTTP/1.1 request (manual parser)
        │     └── Method  Path  Headers  Body
        │
        ├── Router.match(method, path)
        │     ├── Exact match: "/users" → handler
        │     ├── Parameterized: "/users/:id" → handler with params
        │     └── 404 → built-in not_found handler
        │
        ├── CSRF check (if enabled)
        │
        ├── handler(request) → response
        │
        └── write HTTP/1.1 response to TcpStream
```

**Key characteristics:**
- Zero external HTTP dependencies — hand-written TCP server
- Per-connection thread (one thread per request)
- Manual HTTP/1.1 request parsing (no hyper/actix)
- Path parameter extraction via `:param` syntax

### 9.2 Two-Tier Routing

The framework has **two parallel routing systems** that merge at boot time:

```
Compile-time / Setup-time:
  route::register("/users", Method::GET)
  route::register_handler("/users", handler_fn)

Boot-time (serve):
  Router::new()                         → empty router
    .get("/health", health_handler)     → inline registration
    .routes(|r| r.post("/data", ...))   → builder closure
  
  router.extend_from_registry()         → merge route::registry into router
  
  final: Router with all routes
```

The `RouteRegistry` module holds metadata about routes (definitions, methods). The `Router` (in `server`) holds executable handlers. `extend_from_registry()` bridges the two at boot time.

### 9.3 WebSocket Support

```
WsRouter
  ├── .ws("/chat", handler)
  └── .ws_with_config("/chat", config, handler)
        │
        └── attach(router)
              └── Intercepts HTTP upgrade requests
                    ├── Validate Upgrade header
                    ├── SHA-1 handshake (hand-rolled)
                    ├── Base64 accept key (hand-rolled)
                    └── Upgrade → WebSocket frame loop
```

WebSocket implementation is fully self-contained:
- Hand-rolled SHA-1 (`ws/sha1.rs`)
- Hand-rolled Base64 (`ws/base64.rs`)
- Frame encode/decode (opcode, mask, length, payload)
- Ping/pong keepalive

---

## 10. Configuration System

### 10.1 Environment Detection

```rust
enum Environment {
    Local,    // development machine
    Dev,      // shared dev server
    Staging,  // pre-production
    Prod,     // production
    Testing,  // test suite
}
```

Auto-detected or set via `APP_ENV` environment variable. Used for:
- Config file selection (`config.json` + `config.{env}.json`)
- Debug mode activation
- Environment-specific logging

### 10.2 Config Loading Chain

```
config_init("path/to/config")
  │
  ├── Read config.json        → base config
  ├── Read config.{env}.json  → override + merge
  │
  └── ConfigRepository {
        data: HashMap<String, ConfigValue>,
        env: Environment,
      }
```

Config values are accessed via the global `config()` function or through the `Config` facade. Supports nested key access via dot notation: `config("database.connections.pg.host")`.

### 10.3 .env Loading

```
load_env() / load_env_auto()
  │
  ├── Read .env file
  ├── Parse KEY=VALUE lines
  └── Set into std::env (process environment)
```

Available via `env()`, `env_int()`, `env_bool()` functions with default value support.

---

## 11. Plugin System (Gems)

### 11.1 Architecture

```
GemRegistry
  │
  ├── register(gem: impl GemFacade)
  │
  └── before_build_all()
        │
        └── For each registered gem:
              ├── gem.before_build() → process, validate
              └── gem.after_build()  → finalize, cleanup
```

### 11.2 Gem Types

```rust
trait GemFacade {
    fn meta(&self) -> &GemMeta;
    fn before_build(&self) -> Result<()> { Ok(()) }
    fn after_build(&self) -> Result<()> { Ok(()) }
}

pub enum GemKind {
    Integration, Platform, Database, Auth, DevTool, Theme,
    Custom(&'static str),
}
```

### 11.3 Example: TailwindCSS Gem

The `viontin-gem-tailwind` crate implements `GemFacade` to:
1. Read `tailwind.config.toml` from project root
2. Invoke the Tailwind CLI via `tailwind-rs-core`
3. Generate compiled CSS into the output directory
4. Hook into `before_build()` phase

---

## 12. ORM Architecture

### 12.1 Optional ORM — No Lock-In

Viontin Framework has **no built-in ORM dependency**. The framework provides lightweight db types (`framework::db`) and the standalone `viontin-orm` crate is completely optional.

**Three choices:**

| Approach | How | Use Case |
|----------|-----|----------|
| **Built-in** | `framework::db` (always available) | Simple queries, no ORM needed |
| **Standalone ORM** | `viontin-orm = { path = "..." }` | Full-featured query builder, standalone |
| **Via framework** | `features = ["orm"]` on `viontin` | Framework + ORM convenience |

```
┌──────────────────────────────────────┐
│  viontin-orm (standalone, optional)  │
│                                      │
│  QueryBuilder (Laravel Eloquent-     │
│    style, no model required)         │
│  Schema  Blueprint  Migration        │
│  DatabaseType  DriverCapabilities    │
│  NoSqlConnection  DriverRegistry     │
├──────────────────────────────────────┤
│  Driver crates (pg, mysql, sqlite)   │
│  (each depends on viontin-orm only)  │
└──────────────────────────────────────┘
```

### 12.2 Driver Selection

Drivers implement `Connection` and `ConnectionPool` traits from `viontin-orm`. The ORM provides `QueryBuilder`, `Schema`, and `Migration` on top. Switching databases means changing the driver crate in `Cargo.toml` and updating the connection config. No model/active-record layer — use the query builder directly to avoid architecture lock-in.

---

## 13. Error Handling

### 13.1 Core Error Types

```rust
#[derive(Debug, Clone, thiserror::Error)]
pub enum FrameworkError {
    #[error("{0}")]
    Internal(String),
}
```

`FrameworkError` is the unified error type across all framework modules. It currently has a single `Internal` variant wrapping string messages — planned to be expanded with typed variants (`Config`, `Io`, `Parse`, `Validation`, `Authentication`, `NotFound`, etc.).

```rust
pub type Result<T> = std::result::Result<T, FrameworkError>;
```

### 13.2 HTTP Error Handling (Planned)

HTTP-specific error types (`HttpError`, `ErrorReport` with actionable solutions) are planned for future releases but not yet implemented. Currently, error responses are constructed manually:

```rust
fn handler(_req: Request) -> Response {
    // Returns a 500 page
    Response::html("<h1>Error</h1>").status(StatusCode::SERVER_ERROR)

    // Returns a 404 page
    Response::html("<h1>Not Found</h1>").status(StatusCode::NOT_FOUND)
}
```

---

## 14. Debugging & Profiling

| Tool | Function | Description |
|------|----------|-------------|
| `dump()` | `dump!(expr)` | Print variable with file:line label |
| `dd()` | `dd!(expr)` | Dump and exit (die) |
| `Profiler` | `Profiler::new()` | Manual timing: `.start("label")` / `.end("label")` |
| `benchmark()` | `benchmark(fn, iterations)` | Measure execution time |

Debug mode is activated automatically when `Environment::Local` or `APP_DEBUG=true` is detected.

---

## 15. Architectural Testing

### 15.1 ArchRule System

```rust
trait ArchRule {
    fn name(&self) -> &str;
    fn check(&self, target: &str) -> ArchResult;
}
```

**Built-in rules:**

| Rule | Purpose |
|------|---------|
| `IsPascalCase` | Enforce PascalCase naming |
| `IsCamelCase` | Enforce camelCase naming |
| `IsSnakeCase` | Enforce snake_case naming |
| `IsKebabCase` | Enforce kebab-case naming |
| `DoesNotDependOn("mod")` | Forbid importing a module |
| `MustDependOn("mod")` | Require importing a module |
| `EndsWith("suffix")` | Enforce name suffix |
| `StartsWith("prefix")` | Enforce name prefix |

### 15.2 ArchChecker

```rust
let mut checker = ArchChecker::new();
checker.add(IsPascalCase);
checker.add(DoesNotDependOn::new("deprecated_module"));
checker.add(EndsWith::new("Controller"));

let result = checker.check("MyController");
// result.passed() → bool
// result.errors() → Vec<&ArchFinding>
// result.has_errors() → bool

let result = checker.check_all(vec!["ModA", "ModB", "ModC"]);
print_arch_result(&result);
```

Errors are categorized by `ArchSeverity` (Error or Warning) and can be printed in a human-readable format via `print_arch_result()`.

---

## 16. Security Model

| Feature | Implementation | Notes |
|---------|---------------|-------|
| **CSRF** | Token-based, session-backed | `CsrfManager` generates/validates per-session tokens |
| **Authentication** | `AuthGuard` trait | Built-in: `BasicGuard` (HTTP Basic), `SessionGuard` (session-backed) |
| **Session** | Session-backed state | `MemorySession` (dev), `FileSession` (file-backed) |
| **Encryption** | `Encrypter` trait | `SimpleEncrypter` is XOR-based — dev only. Production should use AES via gems |
| **Hashing** | `Hasher` trait | `SimpleHasher` for dev. Production-grade hashing via gems |
| **WebSocket** | Origin validation | Basic origin checking in handshake |

---

## 17. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Zero external HTTP dependencies** | Full control over protocol, no dependency bloat, educational clarity |
| **Hand-rolled SHA-1, Base64** | No external crypto deps for WebSocket handshake |
| **TypeId-based DI container** | Type-safe, no string keys, no runtime downcasting errors |
| **Two-phase boot (register → boot)** | Ensures all providers are registered before any reads dependencies |
| **Facade pattern for drivers** | Clean API for users, swappable backends, zero-cost abstraction |
| **Laravel-inspired CLI signatures** | Familiar ergonomics for web developers entering Rust |
| **Regex-based arch checking** | Simple, no proc-macros, no codegen, no external parser |
| **JSON config with env overlay** | Familiar pattern, zero additional schema definitions needed |
| **Per-connection thread model** | Simplicity over async complexity at this stage of the framework |
| **Separate ORM crate family** | Users pay only for what they use — no SQL deps in framework core |

---

## 18. Example: Full Boot Sequence

```rust
use viontin::{boot, domain, html};
use viontin_gem_tailwind::Tailwind;

fn main() {
    boot()
        .provider(MyDatabaseProvider)      // register service provider
        .command(MyCustomCommand)          // register CLI command
        .gem(Tailwind::load())             // register TailwindCSS plugin
        .get("/", |_req| Response::html(html!("pages/index.html")))
        .post("/submit", submit_handler)
        .ws("/chat", ChatHandler)
        .serve("127.0.0.1:3000");
}
```

This single sequence:
1. Initializes the DI container with 5 built-in providers + user provider
2. Registers CLI commands for `viontin my-command` usage
3. Registers TailwindCSS gem for build-time CSS processing
4. Wires HTTP routes (GET, POST) and WebSocket endpoint
5. Starts the TCP server on port 3000

---

## 19. Appendix: Quick Reference

### 19.1 Module Index

| Module | Layer | Dependencies |
|--------|-------|-------------|
| `app` | Core | config, env, log, queue, events |
| `auth` | Core | support::hash |
| `cache` | Runtime | — |
| `cli` | Runtime | — |
| `collection` | Types | — |
| `config` | Core | env |
| `csrf` | Runtime | session, support |
| `db` | Types/Runtime | — |
| `debug` | Runtime | — |
| `domain` | Runtime | — |
| `encryption` | Core | support |
| `env` | Types | — |
| `error` | Types | — |
| `errors` | Runtime | http, error |
| `events` | Types/Core | — |
| `fs` | Runtime | — |
| `gem` | Runtime | validator, semver |
| `http` | Types | — |
| `lang` | Runtime | — |
| `log` | Types/Core | — |
| `mail` | Types/Runtime | — |
| `notif` | Types/Runtime | mail |
| `page` | Types | — |
| `path` | Runtime | — |
| `queue` | Types/Core | — |
| `rate` | Types/Runtime | cache |
| `route` | Types/Runtime | http, server |
| `schedule` | Types/Runtime | — |
| `semver` | Types | — |
| `server` | Runtime | http |
| `session` | Types/Runtime | — |
| `storage` | Types/Runtime | — |
| `support` | Types | — |
| `testing` | Types/Runtime | — |
| `validator` | Types/Core | — |
| `ws` | Runtime | http, server |

### 19.2 Crate Dependency Table

```
viontin (meta-crate)
  ├── viontin-framework  (serde, serde_json, thiserror, glob)
  ├── viontin-tui        (viontin-framework, crossterm, terminal_size, unicode-width)
  ├── viontin-cli        (viontin-tui, viontin-framework, notify, regex)
  ├── viontin-gems       (viontin-framework, linkme)
  ├── [optional] viontin-orm     (standalone, no framework dependency)
  ├── [optional] viontin-orm-pg  (viontin-orm only)
  ├── [optional] viontin-orm-mysql (viontin-orm only)
  └── [optional] viontin-orm-sqlite (viontin-orm only)
```
