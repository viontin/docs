# Viontin

> **Experimental Project** — This is an experimental project under active development. APIs are unstable, documentation is incomplete, and breaking changes may occur without notice. Not recommended for production use.

**Zero to One, Scale-up Easily** — from prototype to production fleet on one platform.

## Documentation Index

| Category | Documents |
|----------|-----------|
| **Getting Started** | [getting-started](getting-started), [installation](installation), [philosophy](philosophy) |
| **Architecture** | [architecture](architecture), [architecture-patterns](architecture-patterns), [domain-driven](domain-driven) |
| **App Types** | [web-app](web-app), [term-app](term-app), [native-app](native-app) |
| **Core** | [app-boot](app-boot), [providers](providers), [config](config), [environment](environment), [error-handling](error-handling), [path](path), [support](support) |
| **HTTP** | [http](http), [server](server), [async](async), [routing](routing), [middleware](middleware), [csrf](csrf), [websocket](websocket) |
| **Data** | [database](database), [orm](orm), [model](model), [entity](entity), [repository](repository), [collection](collection), [pagination](pagination), [semver](semver) |
| **Security** | [auth](auth), [hashing](hashing), [encryption](encryption), [session](session) |
| **Messaging** | [events](events), [listeners](listeners), [queue](queue), [mail](mail), [notification](notification), [schedule](schedule) |
| **Performance** | [cache](cache), [rate-limit](rate-limit) |
| **i18n** | [localization](localization) |
| **Architecture** | [architecture](architecture), [architecture-patterns](architecture-patterns), [domain-driven](domain-driven), [service](service), [controller](controller) |
| **Validation** | [validation](validation) |
| **Files** | [filesystem](filesystem), [storage](storage) |
| **Dev Tools** | [logging](logging), [debugging](debugging), [testing](testing), [cli](cli), [tui](tui), [known-issues](known-issues) |
| **Extensions** | [gems-system](gems-system), [gems/inertia](gems/inertia), [gems/tailwind](gems/tailwind), [gems/gem-creator](gems/gem-creator) |
| **Standalone** | [viontest](viontest) |

### Feature Flags

| Feature | Cargo Flag | Description |
|---------|-----------|-------------|
| Async server | `async` | Tokio-based async HTTP server |
| Domain-Driven Design | `domain` | DDD building blocks (Domain, AggregateRoot, Repository) |
| Viontin ORM | `orm` | Integrate `viontin-orm` (standalone ORM crate) |

**No vendor lock-in:** The framework works without `viontin-orm`. Use it, or any other ORM, or none at all.

Viontin is a full-stack Rust application framework for building web services, CLI tools, terminal applications, and batch processing systems — all within a single, cohesive platform. It provides HTTP server, ORM, plugin system (Gems), background job processing, mail, notifications, caching, real-time TUI toolkit, architectural enforcement, and more.

---

## Project Structure

```
viontin/
├── repos/
│   ├── framework/          # Core framework monorepo
│   │   └── crates/
│   │       ├── viontin/     # Meta-crate: re-exports everything as `viontin`
│   │       ├── framework/   # Core framework: types, traits, implementations
│   │       ├── cli/         # CLI tool: 42 commands, zero cargo dependency at runtime
│   │       └── tui/         # TUI toolkit: interactive prompts, ANSI styling
│   ├── gems/               # Plugin system (Gems)
│   │   └── crates/
│   │       ├── viontin-gems/           # Gem registry & WASM plugin loader
│   │       └── viontin-gem-tailwind/   # TailwindCSS build-time integration
│   └── orm/                # Multi-driver ORM
│       └── crates/
│           ├── viontin-orm/            # ORM core: Model, Schema, Migration, Relations
│           ├── viontin-orm-pg/         # PostgreSQL driver
│           ├── viontin-orm-mysql/      # MySQL driver
│           └── viontin-orm-sqlite/     # SQLite driver
├── examples/
│   ├── viontin-zero/       # Minimal starter project
│   └── viontin-alpha/      # Feature demo with TailwindCSS + serde
├── docs/
│   └── README.md           ← this file
└── scripts/
    └── install.sh          # CLI installer script
```

---

## Platform Architecture

```
                     ┌──────────────────────────────────────────────┐
                     │               viontin (meta-crate)            │
                     │      re-exports everything for end users       │
                     └──────────────────────────────────────────────┘
                                        │
              ┌─────────────────────────┼─────────────────────────────┐
              │                         │                             │
              ▼                         ▼                             ▼
   ┌──────────────────┐   ┌──────────────────────┐   ┌──────────────────┐
   │  viontin_framework│   │  viontin-cli          │   │  viontin-tui     │
   │  (core library)   │   │  42 commands           │   │  Terminal UI     │
   └──────────────────┘   └──────────────────────┘   └──────────────────┘
              │
   ┌──────────┼──────────┐
   │          │          │
   ▼          ▼          ▼
 ┌──────┐ ┌──────┐ ┌──────────┐
 │ types│ │ core │ │ runtime  │
 │(traits│ │(impl)│ │(servers, │
 │&value)│ │      │ │ drivers) │
 └──────┘ └──────┘ └──────────┘
```

### Three-Layer Platform Architecture

**1. Types & Traits** — Pure interfaces with no runtime dependencies. These define the contracts that the rest of the platform builds upon:

- HTTP: `Request`, `Response`, `StatusCode`, `Method`, `Headers`, `Uri`, `Cookie`
- Security: `AuthGuard`, `AuthUser`, `SessionDriver`
- Data: `Collection`, `Page`, `Version` (semver), `Connection`, `ConnectionPool`
- Messaging: `Event`, `Listener`, `Job`, `Mailer`, `Notification`, `ScheduledJob`
- Performance: `CacheDriver`, `RateLimiter`
- Infrastructure: `LogChannel`, `StorageDriver`, `Validator`, `Encrypter`

**2. Core Implementations** — Concrete implementations of those traits:

- `Application` — DI container, service providers, bootstrapping
- `ConfigRepository` — environment-aware configuration loading
- `Logger` / `StdoutLog` — structured logging
- `BasicGuard` / `SessionGuard` — authentication backends
- `EventDispatcher` — pub/sub event system
- `SyncQueue` — synchronous queue driver
- `SimpleEncrypter` — AES-based encryption
- `ValidatorGroup` — grouped input validation

**3. Runtime** — Executable infrastructure:

- HTTP: `Router`, TCP `Server`, `Handler` type, `RouteRegistry`, CSRF protection
- Caching: `MemoryCache`, `FileCache`, `NullCache`
- Sessions: `MemorySession`, `FileSession`
- Storage: `LocalStorage`, `MemoryStorage`
- Mail: `LogTransport`, `ArrayTransport`
- Notifications: `MailChannel`, `DatabaseChannel`
- Database: `QueryBuilder` (fluent SQL construction)
- Rate Limiting: `TokenBucketLimiter`
- Scheduling: cron-based `Scheduler`
- i18n: JSON-based translations with locale fallback
- Debugging: `dump`, `dd`, `Profiler`, `benchmark` utilities
- Error Handling: `HttpError`, `ErrorReport`, error page rendering
- Architecture: `ArchChecker`, domain boundary analysis

---

## Domain-Driven Architecture

Beyond the modular platform, Viontin enforces architectural integrity at the application level through domain boundaries.

### Domains

A domain is a bounded context that groups related models, routes, events, and business logic. Each domain declares which other domains it may depend on:

```rust
use viontin::Domain;

pub const DEFINITION: Domain = Domain::new("billing")
    .allows(&["order", "payment"]);
```

### CLI Commands

| Command | Purpose |
|---------|---------|
| `viontin make:domain <name>` | Scaffold a new domain module |
| `viontin check --arch` | Scan for cross-domain import violations |
| `viontin inspect --domains` | Visualize domain structure and dependencies |

### Architecture Testing Rules

```rust
use viontin_framework::testing::*;

let mut checker = ArchChecker::new();
checker.add(IsPascalCase);           // enforce naming convention
checker.add(DoesNotDependOn::new("old_module"));  // forbid imports
checker.add(EndsWith::new("Controller"));          // enforce suffix

let result = checker.check("MyModule");
```

Built-in rules: `IsPascalCase`, `IsCamelCase`, `IsSnakeCase`, `IsKebabCase`, `DoesNotDependOn`, `MustDependOn`, `EndsWith`, `StartsWith`.

---

## Plugin System (Gems)

Gems extend the framework at build time via plugin hooks:

- **Gem Registry** — central registry for gem discovery and lifecycle management
- **TailwindCSS Gem** — processes CSS at build time, integrated as a gem
- **Inertia Gem** — InertiaJS SPA bridge with automatic middleware registration
- **Webview Gem** — native desktop window via `wry` + `tao`

> **Planned:** WASM-based plugin loading (`DynamicGem`) for dynamically loading gems compiled to WebAssembly.

---

## ORM Architecture

Multi-driver design with a standalone core (no framework dependency):

| Crate | Role |
|-------|------|
| `viontin-orm` | Core ORM: `QueryBuilder`, `Schema`, `Migration`, `Connection` traits, `Value`/`Row` types |
| `viontin-orm-pg` | PostgreSQL driver (stub) |
| `viontin-orm-mysql` | MySQL driver (stub) |
| `viontin-orm-sqlite` | SQLite driver (stub) |

Each driver implements the core traits, allowing applications to switch databases by changing a single configuration value. No active-record Model layer — use the query builder directly to avoid architecture lock-in.

---

## CLI Reference

The `viontin` CLI provides 42 commands across these categories:

| Category | Commands |
|----------|---------|
| **Project** | `new`, `init`, `dev`, `build`, `run`, `check`, `test`, `add`, `clean` |
| **Make** | `make:controller`, `make:middleware`, `make:model`, `make:route`, `make:command`, `make:event`, `make:job`, `make:mail`, `make:notification`, `make:query`, `make:module`, `make:service`, `make:repository` |
| **Architecture** | `make:domain`, `make:view` |
| **Inspect** | `inspect` (with `--models`, `--routes`, `--commands`, `--events`, `--domains`) |
| **Check** | `check` (with `--arch`) |
| **Cargo Management** | `doc`, `fix`, `bench`, `tree`, `package`, `metadata` |
| **Publishing** | `publish`, `update`, `install`, `uninstall`, `search` |
| **Code Quality** | `fmt`, `clippy` |

> **Planned:** `make:migration`, `make:listener`, `make:validator`, `make:service`, dedicated `gem:install/list/remove`, and interactive scaffolding wizards are on the roadmap.

All commands run with zero `cargo` dependency at runtime, except for build-related commands (`build`, `run`, `test`, `check`, `clean`, `doc`, `fix`, `bench`, `package`, `publish`, `update`, `install`, `uninstall`, `search`, `fmt`, `clippy`) which delegate transparently to `cargo`.

---

## TUI Toolkit

The TUI crate provides building blocks for interactive terminal applications:

| Module | Contents |
|--------|----------|
| `tui` | `Command` trait, `Kernel` (event loop), `Input`, `Output`, `ExitCode` |
| `tui-prompts` | Interactive prompts: `text`, `select`, `confirm`, `password` |
| `tui-styling` | ANSI styling: bold, dim, italic, underline, colors (foreground/background), 256-color and truecolor support |
| `tui-validator` | CLI argument signature parsing and validation |

---

## Examples

| Example | Description |
|---------|-------------|
| `viontin-zero` | Minimal starter — bare `viontin` dependency, `Hello, world!` |
| `viontin-alpha` | Feature demo — includes `viontin-gem-tailwind`, `serde`, `serde_json` |

---

## Quick Start

```bash
cargo install viontin
viontin new my-app
cd my-app
viontin dev
```

## Known Issues

See [known-issues](known-issues) for a comprehensive list of bugs, limitations, and technical debt.

## License

Open source software.
