# Viontin Architecture
> Last updated: 2026-05-25

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

**Version:** 0.1.0
**Tagline:** Zero to One, Scale-up Easily

---

Viontin is a **cloud-native Rust platform** for building microservices, APIs, CLI tools, distributed systems, and mixed-mode binaries — all within a single cohesive framework. This document defines the architectural layering that makes this possible.

---

## 1. Layering Philosophy

Viontin is not a monolithic framework. It is a **stack of layers**, each with clear responsibilities and dependency direction. Every layer depends only on layers below it — never upward.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APPLICATION (your code)                           │
│  routes, controllers, services, models, domains, commands            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              APPLICATION INFRASTRUCTURE KERNEL                │  │
│  │  Boot Lifecycle · DI Container · Service Providers · Facades  │  │
│  │  Config System · Error Handling · Logging · CLI Kernel        │  │
│  │  Middleware Chain · Event System · Scheduling                  │  │
│  └───────────────────────────┬───────────────────────────────────┘  │
│                              │                                      │
│  ┌───────────────────────────▼───────────────────────────────────┐  │
│  │                    RUNTIME EXECUTION                          │  │
│  │  HTTP Server · Router · WebSocket · File System · i18n        │  │
│  │  Cache Drivers · Session Drivers · Storage Drivers            │  │
│  │  Mail Transports · Notification Channels · Queue Workers      │  │
│  └───────────────────────────┬───────────────────────────────────┘  │
│                              │                                      │
│  ┌───────────────────────────▼───────────────────────────────────┐  │
│  │                    CONTRACT LAYER (Traits & Types)            │  │
│  │  Auth · Cache · Collection · Config · Database · Env          │  │
│  │  Events · Http · Log · Mail · Notif · Page · Queue            │  │
│  │  Rate · Route · Schedule · Semver · Session · Storage         │  │
│  │  Support · Validator · Testing · Error                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│  ┌───────────────────────────▼───────────────────────────────────┐  │
│  │                    CORE FOUNDATION                            │  │
│  │  viontin-core: Shared contracts, types, FrameworkError        │  │
│  │  serde · thiserror · serde_json                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer Definitions

| Layer | Role | Depends On |
|-------|------|-----------|
| **Core Foundation** | Shared types, error types, value objects | Rust std + serde |
| **Contract Layer** | Pure traits & interfaces — zero runtime | Core Foundation |
| **Runtime Execution** | Concrete implementations & drivers | Contract Layer |
| **Application Infrastructure Kernel** | Application lifecycle, DI, config, boot | Runtime Execution |
| **Application** | User code — routes, controllers, services, domains | Infrastructure Kernel |

---

## 2. Foundation Layer — Core Foundation

The base of the entire platform. Only depends on `serde`, `serde_json`, and `thiserror`.

**Crate:** `viontin-core`

```
viontin-core
├── error.rs          → FrameworkError (unified error type)
├── http.rs           → Request, Response, StatusCode, Method, Headers, Uri, Cookie
├── connection.rs     → Connection, ConnectionPool, Value, Row, DbConfig
├── entity.rs         → Entity trait (business data container)
├── log.rs            → Level, LogEntry, LogChannel trait
├── debug.rs          → Debug trait extensions
├── pair.rs           → Key-value pair types
└── lib.rs            → Re-exports
```

```rust
// Single error type for the entire platform
#[derive(Debug, Clone, thiserror::Error)]
pub enum FrameworkError {
    #[error("{0}")]
    Internal(String),
    // Planned: Config, Io, Parse, Validation, Authentication, NotFound, ...
}

pub type Result<T> = std::result::Result<T, FrameworkError>;
```

**Key design decision:** Every crate in the ecosystem depends on `viontin-core` for shared types — no circular dependencies, no duplicated definitions.

---

## 3. Contract Layer — Traits & Types

Pure interfaces with **zero runtime implementation**. These modules define WHAT the platform can do, not HOW.

**Crate:** `viontin-framework` (types module group)

```
Layer Boundary Rules:
- NO runtime dependencies (no std::net, no filesystem I/O)
- NO concrete implementations
- ONLY traits, enums, structs, type aliases
- Every trait has ≤6 methods (single responsibility)
```

### Module Inventory

| Module | Key Contracts | Methods |
|--------|--------------|---------|
| `auth` | `AuthGuard`, `AuthUser`, `AuthResult` | authenticate, user |
| `cache` | `CacheDriver` | get, set, has, forget, clear |
| `collection` | `Collection<T>` | map, filter, reduce, chunk, unique, sum, avg |
| `config` | `ConfigValue` (String/Int/Float/Bool/Array/Object/Null) | as_str, as_int, as_bool, get, set |
| `database` | `Connection`, `ConnectionPool`, `Value`, `Row` | query, execute, prepare |
| `env` | `Environment` enum | Local, Dev, Staging, Prod, Testing |
| `events` | `Event`, `Listener`, `Subscriber`, `Subscribable` | handle, subscribe |
| `http` | `Request`, `Response`, `StatusCode`, `Method`, `Headers`, `Uri`, `Cookie` | parse, serialize |
| `log` | `Level`, `LogEntry`, `LogChannel` | log, flush |
| `mail` | `Mailer`, `Envelope`, `Attachment` | send, queue, later |
| `notif` | `Notification`, `Notifiable`, `Channel` | send, route |
| `page` | `Page<T>`, `PaginationLinks` | paginate, links |
| `queue` | `Job`, `Driver` | push, pop, process |
| `rate` | `RateLimiter` | attempt, hit, tooManyAttempts |
| `route` | `RouteDefinition`, `RouteMethod` | match, compile |
| `schedule` | `ScheduledJob`, `cron_matches()` | due, run |
| `semver` | `Version`, `VersionReq`, `Meta`, `Compatibility` | parse, compare, compatible |
| `session` | `SessionDriver` | get, set, has, forget, flush |
| `storage` | `Driver` | get, put, delete, exists, url |
| `support` | `Hasher`, `Encrypter`, `SimpleHasher` | hash, encrypt, verify |
| `testing` | `ArchRule`, `ArchResult`, `ArchSeverity`, `ArchFinding` | check, name |
| `validator` | `Validator`, `Outcome`, `Finding`, `Severity`, `Context` | validate, passes |

---

## 4. Runtime Execution Layer — Drivers & Servers

Concrete implementations of the Contract Layer traits. Every trait has at least one built-in implementation plus a facade.

**Crate:** `viontin-framework` (runtime module group)

### 4.1 Facade Pattern Architecture

Every subsystem follows the same pattern:

```
┌──────────────────────────────────────────────────────────────┐
│                       FACADE                                  │
│  pub struct Cache { driver: Box<dyn CacheDriver> }           │
│  impl Cache {                                                │
│      pub fn memory() -> Self { Self::new(MemoryCache) }      │
│      pub fn file(path) -> Self { Self::new(FileCache) }      │
│      pub fn null() -> Self { Self::new(NullCache) }          │
│                                                              │
│      pub fn get(&self, key) -> Option<Value> {               │
│          self.driver.get(key)   // delegates                 │
│      }                                                       │
│  }                                                           │
└──────────────────────────────────────────────────────────────┘
```

The facade IS the public API. Users never interact with drivers directly.

### 4.2 Built-in Drivers

| Facade | Trait | Built-in Drivers |
|--------|-------|-----------------|
| `Cache` | `CacheDriver` | `MemoryCache`, `FileCache`, `NullCache` |
| `Session` | `SessionDriver` | `MemorySession`, `FileSession` |
| `Storage` | `Driver` (storage) | `LocalStorage`, `MemoryStorage` |
| `Mail` | `Mailer` | `LogTransport`, `ArrayTransport` |
| `Queue` | `Driver` (queue) | `SyncQueue` |
| `Notif` | `NotificationChannel` | `MailChannel`, `DatabaseChannel` |
| `Auth` | `AuthGuard` | `BasicGuard`, `SessionGuard` |

### 4.3 HTTP Server

**Zero external HTTP dependencies.** Hand-written TCP server on `std::net::TcpListener`.

```
Server (TcpListener)
│
├── accept loop (per-connection thread)
│     │
│     ├── parse HTTP/1.1 request (manual parser)
│     │     └── Method · Path · Headers · Body
│     │
│     ├── Router.match(method, path)
│     │     ├── Exact: "/users" → handler
│     │     ├── Parameterized: "/users/:id" → handler + params
│     │     └── 404 → built-in handler
│     │
│     ├── [Middleware Chain] → request/response pipeline
│     │
│     ├── handler(request) → response
│     │
│     └── write HTTP/1.1 response to TcpStream
```

**Key characteristics:**
- No `hyper`, `actix`, or `tokio` dependency
- Per-connection threading model (one OS thread per request)
- Path parameter extraction via `:param` syntax
- Built-in `not_found` and `method_not_allowed` handlers

### 4.4 WebSocket

Fully self-contained WebSocket implementation:

```
WsRouter
├── .ws("/chat", handler)
└── attach(router)
      └── Intercepts HTTP upgrade requests
            ├── Validate Upgrade header
            ├── SHA-1 handshake (hand-rolled, ws/sha1.rs)
            ├── Base64 accept key (hand-rolled, ws/base64.rs)
            └── Upgrade → WebSocket frame loop
                  ├── Frame encode/decode (opcode, mask, length, payload)
                  └── Ping/pong keepalive
```

### 4.5 Runtime Module Map

| Module | Key Implementations |
|--------|-------------------|
| `server` | `Router`, `Server` (TCP), per-connection thread pool, graceful shutdown |
| | Submodules: `server` (Server + connection handler) |
| `routing` | `RouteRegistry`, global `register()`/`register_handler()`/`finalize()`, `RouteProvider` |
| `caching` | `MemoryCache`, `FileCache`, `NullCache`, `Cache` facade |
| | Ergonomic constructors: `Cache::memory()`, `Cache::file(p)`, `Cache::null()` |
| `sessions` | `MemorySession`, `FileSession`, `Session` facade |
| `storage` | `LocalStorage`, `MemoryStorage`, `Storage` facade |
| `mail` | `LogTransport`, `ArrayTransport`, `SmtpTransport` (cfg), `Mail` facade |
| | Submodules: `log`, `array`, `smtp` (cfg) |
| `notif` | `MailChannel`, `DatabaseChannel`, `Notif` facade |
| `db` | `QueryBuilder` (fluent SQL: select/where/join/orderBy/groupBy/limit/offset) |
| `fs` | `read()`, `write()`, `copy()`, `delete()`, `ensure_dir()`, `TempDir`, `FileInfo`, `find_files()` |
| `csrf` | `CsrfManager`, `CsrfConfig` (token generation/validation, session-backed) |
| `rate` | `TokenBucketLimiter` (backed by Cache driver) |
| `schedule` | `Scheduler` (cron expression engine, job runner) |
| `lang` | `JsonTranslator` (JSON-based with locale fallback), `trans()`/`choice()`/`locale()` |
| `debug` | `dump()`, `dd()`, `Profiler`, `benchmark()`, `memory_usage()` |
| `errors` | `HttpError`, `ErrorReport`, error page rendering |
| `ws` | `WebSocketHandler`, `WsRouter`, `WsServer`, frame encode/decode, handshake |
| | Submodules: `frame`, `sha1`, `base64` |
| `cli` | `Command`, `Kernel`, `Input`, `Output`, `ExitCode`, `Signature`, `ArgDef`, `OptDef`, `Spinner`, `ProgressBar` |
| `middleware` | `CorsMiddleware`, `PanicRecovery`, `RequestId`, `RateLimitMiddleware`, `MiddlewareChain` |
| | Submodules: `cors`, `panic`, `health`, `static_files`, `timeout`, `request_id`, `rate_limit` |

---

## 5. Application Infrastructure Kernel — The Core of Every App

This is the layer that turns a collection of libraries into a **running application**. It is the most important layer for understanding how Viontin works.

```
┌─────────────────────────────────────────────────────────────────────┐
│                  APPLICATION INFRASTRUCTURE KERNEL                    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Boot Lifecycle                                               │   │
│  │  boot() → BootBuilder → .serve() / .run()                    │   │
│  │  Orchestrates the entire startup sequence                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  DI Container (TypeId-based)                                  │   │
│  │  Container::new() → .singleton() → .resolve() → .has()       │   │
│  │  No string keys, no runtime downcasting errors                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Service Provider System                                      │   │
│  │  2-phase boot: 1) register() 2) boot()                       │   │
│  │  5 built-in providers + user providers                        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Configuration System                                         │   │
│  │  .env loading → JSON config → env overlay → dot notation     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Logging Infrastructure                                       │   │
│  │  Logger + LogChannel → global log_info()/log_error()         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Error Handling Infrastructure                                │   │
│  │  FrameworkError → ErrorReport → actionable solutions         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Event System (pub/sub)                                       │   │
│  │  EventDispatcher → Listener → Subscriber                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  CLI Kernel                                                   │   │
│  │  Kernel::run() → Command dispatch → Input/Output → ExitCode  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Middleware Chain                                             │   │
│  │  Middleware trait → request/response pipeline                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Gem Registry & Plugin System                                 │   │
│  │  GemRegistry → GemFacade → before_build / after_build        │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.1 Boot Lifecycle — Detailed Flow

The `boot()` function creates a `Boot` builder and orchestrates the entire startup:

```
boot()
│
├── Initialize base components:
│     ├── Application::new()
│     │     └── Container::new()          → empty TypeId map
│     │     └── 5 built-in providers:
│     │           EnvProvider             → loads .env
│     │           ConfigProvider          → loads config/*.json
│     │           LogProvider             → initializes Logger
│     │           QueueProvider           → initializes SyncQueue
│     │           EventsProvider          → initializes EventDispatcher
│     │
│     ├── Kernel::new()                   → empty CLI command registry
│     ├── Router::new()                   → empty HTTP route table
│     ├── WsRouter::new()                 → empty WebSocket route table
│     ├── GemRegistry::new()              → empty plugin registry
│     └── MiddlewareChain::new()          → empty middleware pipeline
│
├── Builder phase (user code runs here):
│     ├── .provider(P)                   → register service provider
│     ├── .command(C)                    → register CLI command
│     ├── .gem(G)                        → register gem plugin
│     ├── .middleware(M)                 → register middleware
│     ├── .get("/", h) / .post(...)      → register HTTP routes
│     ├── .ws("/", h)                    → register WebSocket routes
│     └── .entry(f) / .serve(a) / .run() → terminal method
│
└── Terminal method triggered:
      │
      ├── 1. gems.before_build_all()
      │     → All plugins run their pre-build hooks
      │     (asset processing, config validation, code generation)
      │
      ├── 2. app.run()
      │     → Phase 1: register() for every provider (in order)
      │     → Phase 2: boot() for every provider (in order)
      │     Order: Env → Config → Log → Queue → Events → User providers
      │
      ├── 3. CLI dispatch (if CLI args detected)
      │     → kernel.run(&args)
      │     → If command matched: execute and exit
      │
      ├── 4. Router finalization
      │     → router.extend_from_registry()
      │     → ws_router.attach(router)
      │     → middleware_chain.apply(router)
      │
      └── 5. Entry point execution
            → If .serve(addr):
                  server.run(addr)  ← blocking TCP listener
            → If .entry(f):
                  f(BootContext)    ← user callback
```

### 5.2 DI Container — TypeId-Based

```rust
pub struct Container {
    bindings: HashMap<TypeId, Box<dyn Any>>,
}
```

**Why TypeId?** No string keys, no typos at runtime, no downcasting errors. Each type is its own key.

```
Container
├── singleton::<T>(value: T)          → store by TypeId::of::<T>()
├── resolve::<T>() -> Option<&T>      → lookup by TypeId::of::<T>()
├── has::<T>() -> bool                → check if TypeId exists
└── remove::<T>()                     → remove binding
```

### 5.3 Service Provider System — Two-Phase Boot

Inspired by Laravel's service container. Every provider implements:

```rust
pub trait ServiceProvider: Debug + Send + Sync {
    fn name(&self) -> &str;                      // unique identifier
    fn register(&self, app: &mut Application) {} // Phase 1: bind services
    fn boot(&self, app: &Application) {}          // Phase 2: initialize
}
```

**Two-phase boot guarantees:**
- Phase 1 runs `register()` for ALL providers
- Phase 2 runs `boot()` for ALL providers (same order)
- This ensures every provider's services are available before any provider tries to resolve them

**Built-in providers (execution order):**

| # | Provider | Name | register() | boot() |
|---|----------|------|-----------|--------|
| 1 | `EnvProvider` | `"env"` | — | Load `.env` into process env |
| 2 | `ConfigProvider` | `"config"` | — | Read `config/*.json`, build `ConfigRepository` |
| 3 | `LogProvider` | `"log"` | — | Initialize global `Logger` with `StdoutLog` |
| 4 | `QueueProvider` | `"queue"` | Bind `SyncQueue` singleton | — |
| 5 | `EventsProvider` | `"events"` | Bind `EventDispatcher` singleton | — |

### 5.4 Configuration System

```
Environment Detection:
  APP_ENV → Environment enum (Local|Dev|Staging|Prod|Testing)
  ↓
Config Loading Chain:
  config_init("path/to/config")
    ├── Read config.json           → base config
    └── Read config.{env}.json     → env-specific overrides (merged)
    ↓
  ConfigRepository {
      data: HashMap<String, ConfigValue>,
      env: Environment,
  }
  ↓
Access Patterns:
  config("database.connections.pg.host")    → dot notation
  config("app.name")                         → global function
  env("DATABASE_URL")                        → .env values
  env_int("PORT")                            → typed access
  env_bool("APP_DEBUG")                      → boolean access
```

### 5.5 Error Handling Infrastructure

```
Application Error Flow:
  FrameworkError (unified error type)
    │
    ├── Internal(String)          → generic errors
    │
    ├── Planned typed variants:
    │     Config, Io, Parse, Validation,
    │     Authentication, NotFound, RateLimit, Csrf
    │
    └── ErrorReport (actionable errors):
          ├── title: String
          ├── detail: String
          ├── solution: Option<String>  → "Try running: viontin make:domain billing"
          ├── file: SourceLocation
          └── hints: Vec<String>
```

### 5.6 Logging Infrastructure

```
LogChannel trait → StdoutLog (built-in)
  │
  ├── log(entry: LogEntry)
  │     └── Level: Debug, Info, Notice, Warning, Error, Critical, Alert
  │
  ├── Global functions:
  │     log_info!("User {} logged in", id)
  │     log_error!("Failed to process payment: {}", err)
  │     log_debug!("SQL: {}", query)
  │     log_warning!("Rate limit hit for IP {}", ip)
  │
  └── Logger struct (wraps LogChannel)
        └── Automatic environment detection:
              Local/Dev → verbose output
              Prod → structured JSON
```

### 5.7 Event System — Pub/Sub

```
EventDispatcher (singleton in container)
  │
  ├── listen::<E, L>()           → register listener for event
  ├── subscribe(subscriber)      → register subscriber (multiple events)
  ├── dispatch(&event)           → publish event to all listeners
  │
  ├── Built-in:
  │     Event trait → Listener trait → Subscriber trait
  │     SyncQueue integration (jobs can be dispatched as events)
  │
  └── Domain events (behind `domain` feature):
        DomainEvent trait → extends Event with domain metadata
        DomainListener → scoped to specific domain
```

### 5.8 CLI Kernel — Command Dispatch

```
Kernel::run(&args)
  │
  ├── Parse CLI args → extract command name + arguments
  ├── Lookup Command by signature name match
  │
  ├── If found:
  │     ├── signature.parse(args) → Input (validated arguments)
  │     ├── command.handle(input, output) → ExitCode
  │     └── return ExitCode
  │
  └── If not found:
        ├── Show "Command not found"
        └── Suggest similar commands (fuzzy match)

Command trait:
  trait Command {
      fn signature(&self) -> &Signature;     // "make:controller {name} {--force}"
      fn description(&self) -> &str;
      fn handle(&self, input: Input, output: Output) -> ExitCode;
  }

Signature syntax (Laravel-inspired):
  "make:controller {name} {--force} {--resource} {--type=default}"
  │              │         │          │            │
  command name   required   flag      flag        option with default
                 argument
```

### 5.9 Middleware Chain

```
Middleware trait:
  trait Middleware: Debug + Send + Sync {
      fn name(&self) -> &str;
      fn handle(&self, req: &mut Request, next: Next) -> Response;
  }

Pipeline execution:
  Request → Middleware1 → Middleware2 → ... → Handler → Response
                                        ↓
                           (any middleware can short-circuit)
```

### 5.10 Gem Registry — Plugin System

```
GemRegistry
  │
  ├── register(gem: impl GemFacade)
  │
  ├── before_build_all()
  │     └── For each gem:
  │           gem.before_build() → process, validate, transform
  │
  └── after_build_all()
        └── For each gem:
              gem.after_build() → finalize, cleanup

GemFacade trait:
  trait GemFacade {
      fn meta(&self) -> &GemMeta;
      fn before_build(&self) -> Result<()> { Ok(()) }
      fn after_build(&self) -> Result<()> { Ok(()) }
  }

GemBinding (auto-wiring):
  impl GemBinding for Inertia {
      fn gem_middlewares(&self) -> Vec<Box<dyn Middleware>> {
          vec![Box::new(InertiaMiddleware::new())]
      }
      // Also: gem_providers(), gem_commands(), gem_routes()
  }

Boot::gem() auto-detects GemBinding and wires everything automatically.
```

---

## 6. Crate Architecture — Platform Layer

### 6.1 Workspace Organization

```
viontin/ (root workspace — NOT a git repo)
│
├── products/
│   ├── viontin/           → Meta-crate, CLI & macros
│   │   └── crates/
│   │       ├── core/      → Foundation Layer (serde, thiserror)
│   │       ├── viontin/   → Facade meta-crate (re-exports everything)
│   │       ├── cli/       → CLI binary (45 commands)
│   │       └── macros/    → Proc macros (#[domain], #[domain_event])
│   │
│   ├── framework/         → Framework implementations
│   │   └── crates/
│   │       ├── framework/ → Contract + Runtime + Infrastructure Kernel
│   ├── tui/               → TUI toolkit (prompts, styling, ANSI) — standalone
│   │
│   ├── gems/              → Plugin system
│   │   └── crates/
│   │       ├── gems/      → Gem registry & loader
│   │       ├── inertia/   → InertiaJS SPA adapter
│   │       ├── tailwind/  → TailwindCSS build-time integration
│   │       └── webview/   → Desktop webview (wry + tao)
│   │
│   ├── orm/               → Standalone ORM (zero framework deps)
│   │   └── crates/
│   │       ├── orm/       → QueryBuilder, Schema, Migration
│   │       ├── pg/        → PostgreSQL driver
│   │       ├── mysql/     → MySQL driver
│   │       └── sqlite/    → SQLite driver (bundled)
│   │
│   ├── viontest/          → Standalone testing framework
│   │   └── crates/
│   │       └── viontest/  → Test runner, assertions, arch testing
│   │
│   ├── ui/                → (Future) Native UI framework
│   └── engine/            → (Future) Game engine
│
├── examples/ (workspace)
│   ├── viontin-zero/      → Minimal starter
│   └── viontin-alpha/     → Feature demo
│
└── docs/                  → Documentation
```

### 6.2 Crate Dependency Graph

```
                    ┌─────────────────────────────────────────────┐
                    │               viontin (meta-crate)           │
                    │     re-exports: core, cli, macros            │
                    │     depends on: framework, gems, tui         │
                    └─────────────────────────────────────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            ▼                      ▼                      ▼
┌──────────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
│  framework    │ │  viontin-tui      │ │  viontin-cli         │
│  All layers:        │ │  Terminal UI      │ │  Binary (45 cmds)   │
│  Contract + Runtime │ │  (standalone)     │ │                      │
│  Infrastructure     │ │  Prompts/styling  │ └──────────────────────┘
└──────────────────────┘ └──────────────────┘
            │
            ├───────────────────────┬───────────────────────┐
            ▼                       ▼                       ▼
┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
│  viontin-macros      │ │  gems         │ │  orm (optional)   │
│  Proc macros         │ │  Plugin registry      │ │  ORM core + drivers  │
│  #[domain] etc.      │ │  WASM loader          │ │  pg/mysql/sqlite     │
└──────────────────────┘ └──────────────────────┘ └──────────────────────┘
```

### 6.3 Feature Flags

| Feature | Cargo Flag | Layer | Description |
|---------|-----------|-------|-------------|
| Async server | `async` | Runtime | Tokio-based async HTTP server |
| Domain-Driven | `domain` | Infra | DDD building blocks, boundary checking |
| ORM integration | `orm` | Runtime | Enable `orm` crate via framework |
| HTTP Client | `http-client` | Runtime | `ureq`-based sync HTTP client |
| Graceful Shutdown | `shutdown` | Infra | SIGTERM/SIGINT handling (default) |
| AES Encryption | `aes` | Runtime | AES-256-GCM via `aes-gcm` |
| SMTP Mail | `smtp` | Runtime | SMTP email transport via `lettre` |

---

## 7. Data Layer — ORM & Storage

### 7.1 Three-Tier Data Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Application Code                                         │
│  Entity/Model structs, Repository pattern, QueryBuilder  │
├──────────────────────────────────────────────────────────┤
│  ORM Core (standalone — zero framework deps)              │
│  QueryBuilder · Schema · Migration · Connection traits    │
├──────────────────────────────────────────────────────────┤
│  Database Drivers                                          │
│  pg (PostgreSQL) · mysql (MySQL) · sqlite (SQLite)        │
└──────────────────────────────────────────────────────────┘
```

### 7.2 Design Decisions

- **No active-record Model layer** — use `QueryBuilder` directly
- **Switch databases by changing one config value** — traits abstract drivers
- **Standalone ORM crate** — use with or without Viontin framework
- **Laravel Eloquent-style query builder** — familiar API for web developers

```rust
// Same code, different databases:
let users = QueryBuilder::table(&conn, "users")
    .where_eq("active", true)
    .order_by("created_at", "DESC")
    .limit(10)
    .get()?;
```

---

## 8. Layers & Application Types

The same infrastructure kernel powers FOUR application types:

### 8.1 Web Server

```rust
boot()
    .get("/", handler)
    .post("/submit", submit)
    .middleware(LoggerMw)
    .provider(DatabaseProvider)
    .serve("127.0.0.1:3000");
```

### 8.2 CLI Tool

```rust
boot()
    .command(DeployCommand)
    .command(RollbackCommand)
    .command(BackupCommand)
    .run_with(|_ctx| {});
```

### 8.3 TUI Application

```rust
boot()
    .command(InteractiveWizard)
    .run_with(|ctx| {
        // ctx.cli() dispatches the TUI command
    });
```

### 8.4 Batch/Daemon Processor

```rust
boot()
    .command(ProcessQueueCommand)
    .provider(QueueWorkerProvider)
    .entry(|ctx| {
        let app = ctx.into_inner();
        loop {
            // process queue jobs forever
        }
    })
    .run();
```

All four share the same DI container, config system, logging, events, and service providers.

---

## 9. Module Dependency Map

```
Infrastructure Kernel Layer:
  app → config, env, log, queue, events
  auth → support::hash
  csrf → session
  encryption → support (Encrypter trait)
  notif → mail (Envelope)
  rate → cache (CacheDriver)
  gem::validator → validator, semver

Runtime Layer:
  route → http (Method), server (Handler)
  route::provider → app (ServiceProvider), route
  server → http (Request, Response, etc.)
  ws → http, server (Router)
  cli::output → log (LogChannel)
  cli::input → cli::command (Signature)
  cli::kernel → cli::command, cli::input, cli::exit, cli::output
  semver::compat → semver::version, semver::constraint
  semver::constraint → semver::version

Standalone Modules (zero internal deps):
  collection · env · fs · page · path · support · debug · testing::arch
```

---

## 10. Key Design Decisions

| Decision | Rationale | Layer |
|----------|-----------|-------|
| **Zero external HTTP deps** | Full protocol control, no bloat, educational clarity | Runtime |
| **Hand-rolled SHA-1 + Base64** | No external crypto for WebSocket handshake | Runtime |
| **TypeId-based DI container** | Type-safe, no string keys, zero downcasting errors | Infra Kernel |
| **Two-phase boot** | All providers register before any reads dependencies | Infra Kernel |
| **Facade pattern for drivers** | Clean API, swappable backends, zero abstraction cost | Runtime |
| **Laravel-inspired CLI signatures** | Familiar ergonomics for web developers | Infra Kernel |
| **Regex-based arch checking** | Simple, no proc-macros, no codegen, no external parser | Infra Kernel |
| **JSON config with env overlay** | Familiar pattern, zero schema definitions | Infra Kernel |
| **Per-connection thread model** | Simplicity over async at this stage | Runtime |
| **Separate ORM crate family** | Pay only for what you use — no SQL in framework core | Data |
| **Minimal proc macros (2 only)** | Fast compilation, no tooling friction | Foundation |
| **Single meta-crate import** | One dependency unlocks entire platform | Platform |
| **Standalone CLI binary** | 45 commands, zero cargo at runtime | Platform |
| **Actionable error reports** | Every error includes `solution` field | Infra Kernel |

---

## 11. Layer Summary

```
Layer                    Crate(s)              Dependencies
─────                    ────────              ────────────
PLATFORM                 viontin (meta-crate)  All layers below
│                        viontin-cli           tui + framework
│                        viontin-macros        syn + quote + proc-macro2
│                        viontin-tui           framework + crossterm
│                        (standalone)
│
├── APPLICATION          Your code             Infrastructure Kernel
│   (routes, controllers, services, domains)
│
├── INFRASTRUCTURE       viontin-framework     Runtime + Contract
│   KERNEL               (app, config, log,
│    Boot, DI, Providers  events, cli, csrf,
│    Config, Errors, Gems rate, schedule, lang,
│    CLI, Middleware,     debug, errors, domain,
│    Events, Scheduling   testing, gem)
│
├── RUNTIME              viontin-framework     Contract
│   EXECUTION            (server, routing,
│    HTTP, WebSocket,     caching, sessions,
│    Cache, Session,      storage, mail, notif,
│    Storage, Mail,       db, fs)
│    Queue, DB, FS
│
├── CONTRACT             viontin-framework     Core Foundation
│   LAYER                (types module group:
│    Traits & Types       auth, cache, config,
│                         database, env, events,
│                         http, log, mail, notif,
│                         page, queue, rate,
│                         route, schedule, semver,
│                         session, storage,
│                         support, validator, testing)
│
├── FOUNDATION           viontin-core          serde + thiserror
│   Shared Types,
│   FrameworkError
│
└── EXTENSIONS (opt)     gems / orm / ui       framework + linkme
    Gems, ORM, Drivers   / pg / mysql / sqlite  / orm
```

---

## 12. Example: Full Boot Sequence

```rust
use viontin::{boot, domain, html};
use viontin_gem_tailwind::Tailwind;

fn main() {
    boot()
        .provider(MyDatabaseProvider)      // Infrastructure: register provider
        .command(MyCustomCommand)          // Infrastructure: register CLI command
        .gem(Tailwind::load())             // Infrastructure: register plugin
        .get("/", |_req| Response::html(include_html!("pages/index.html")))
        .post("/submit", submit_handler)
        .ws("/chat", ChatHandler)
        .serve("127.0.0.1:3000");
}
```

What happens:

```
Foundation   → viontin-core types available
Contract     → traits for HTTP, cache, events, etc. defined
Runtime      → TCP server ready, cache drivers loaded
Infra Kernel → DI container initialized
             → .env loaded (EnvProvider)
             → config/*.json loaded (ConfigProvider)
             → Logger initialized (LogProvider)
             → Queue/Events singletons bound
             → User provider registered & booted
             → CLI commands registered
             → Gem plugins initialized (TailwindCSS processes CSS)
             → Router built, middleware chain attached
             → WebSocket routes attached
Platform     → TCP listener bound to 127.0.0.1:3000
             → Per-connection thread pool ready
```