# Viontin

> **Experimental Project** — This is an experimental project under active development. APIs are unstable, documentation is incomplete, and breaking changes may occur without notice. Not recommended for production use.
> Last updated: 2026-05-25

**Cloud-Native Rust Framework** — build, deploy, and scale microservices, APIs, and CLI tools.

## Documentation Index

| Category | Documents |
|----------|-----------|
| **Getting Started** | [getting-started](getting-started), [installation](installation), [philosophy](philosophy) |
| **Platforms** | [webapp](platforms/webapp), [terminal](platforms/terminal), [desktop](platforms/desktop), [cloud](platforms/cloud) |
| **Core** | [app-boot](core/app-boot), [providers](core/providers), [config](core/config), [environment](core/environment), [error-handling](core/error-handling), [path](core/path), [support](core/support) |
| **HTTP** | [http](http/http), [server](http/server), [async](http/async), [routing](http/routing), [middleware](http/middleware), [csrf](http/csrf), [websocket](http/websocket) |
| **Data** | [database](data/database), [orm](data/orm), [model](data/model), [entity](data/entity), [repository](data/repository), [collection](data/collection), [pagination](data/pagination), [semver](data/semver) |
| **Security** | [auth](security/auth), [hashing](security/hashing), [encryption](security/encryption), [session](security/session) |
| **Messaging** | [events](messaging/events), [listeners](messaging/listeners), [queue](messaging/queue), [mail](messaging/mail), [notification](messaging/notification), [schedule](messaging/schedule) |
| **Performance** | [cache](performance/cache), [rate-limit](performance/rate-limit) |
| **i18n** | [localization](i18n/localization) |
| **Architecture** | [architecture](architecture/architecture), [architecture-patterns](architecture/architecture-patterns), [domain-driven](architecture/domain-driven), [service](core/service), [controller](core/controller) |
| **Validation** | [validation](validation/validation) |
| **Files** | [filesystem](files/filesystem), [storage](files/storage) |
| **Dev Tools** | [logging](dev-tools/logging), [debugging](dev-tools/debugging), [testing](dev-tools/testing), [cli](dev-tools/cli), [tui](dev-tools/tui), [macros](dev-tools/macros), [deployment](dev-tools/deployment), [roadmap](dev-tools/roadmap), [known-issues](dev-tools/known-issues), [comparisons](dev-tools/comparisons) |
| **Extensions** | [gems-system](extensions/gems-system), [gems/inertia](gems/inertia), [gems/tailwind](gems/tailwind), [gems/gem-creator](gems/gem-creator) |
| **Standalone** | [viontest](standalone/viontest) |

### Feature Flags

| Feature | Cargo Flag | Description |
|---------|-----------|-------------|
| Async server | `async` | Tokio-based async HTTP server |
| Domain-Driven Design | `domain` | DDD building blocks (Domain, AggregateRoot, Repository) |
| Viontin ORM | `orm` | Integrate `orm` (standalone ORM crate) |
| HTTP Client | `http-client` | `ureq`-based sync HTTP client for external APIs |
| Graceful Shutdown | `shutdown` | SIGTERM/SIGINT handling (enabled by default) |
| AES Encryption | `aes` | AES-256-GCM encryption via `aes-gcm` crate |
| SMTP Mail | `smtp` | SMTP email transport via `lettre` |

**No vendor lock-in:** The framework works without `orm`. Use it, or any other ORM, or none at all.

Viontin is a **cloud-native Rust framework** for building microservices, APIs, CLI tools, and distributed systems. It provides HTTP server, ORM, plugin system (Gems), background job processing, mail, notifications, caching, real-time TUI toolkit, architectural enforcement, and more — all optimized for cloud deployment.

---

## Project Structure

```
viontin/
├── products/
│   ├── viontin/            # Core + CLI + Facade meta-crate + macros
│   │   └── crates/
│   │       ├── core/        # Shared contracts & types (zero deps: serde, thiserror)
│   │       ├── viontin/     # Meta-crate: re-exports everything
│   │       ├── cli/         # CLI tool: 45 commands, zero cargo dependency
│   │       └── macros/      # Proc macros: #[domain], #[domain_event]
│   ├── framework/          # Framework implementations & patterns
│   │   └── crates/
│   │       └── framework/   # Core library: HTTP, ORM, patterns, infrastructure
│   ├── tui/                # TUI toolkit: prompts, styling, ANSI (standalone)
│   ├── gems/               # Plugin system (Gems)
│   │   └── crates/
│   │       ├── gems/        # Gem registry & plugin loader
│   │       ├── inertia/     # InertiaJS SPA adapter
│   │       ├── tailwind/    # TailwindCSS build-time integration
│   │       └── webview/     # Desktop webview (wry + tao)
│   ├── orm/                # Standalone ORM (zero framework deps)
│   │   └── crates/
│   │       ├── orm/         # ORM core: QueryBuilder, Schema, Migration
│   │       ├── pg/          # PostgreSQL driver
│   │       ├── mysql/       # MySQL driver
│   │       └── sqlite/      # SQLite driver (rusqlite, bundled)
│   ├── viontest/           # Standalone testing framework (zero deps)
│   │   └── crates/
│   │       └── viontest/    # Test runner, assertions, arch testing
│   ├── ui/                 # Future: Native UI framework (standalone)
│   └── engine/             # Future: Game engine (standalone)
├── examples/
│   ├── viontin-zero/       # Minimal starter project
│   └── viontin-alpha/      # Feature demo with TailwindCSS + serde
├── docs/                   # Documentation (separate repo)
└── scripts/
    └── install.sh          # CLI installer script
```

---

## Platform Architecture

```
                      ┌──────────────────────────────────────────────┐
                      │               viontin (meta-crate)            │
                      │      re-exports everything for end users       │
                      │      contains: core, cli, macros              │
                      └──────────────────────────────────────────────┘
                                         │
               ┌─────────────────────────┼─────────────────────────────┐
               │                         │                             │
               ▼                         ▼                             ▼
    ┌──────────────────┐   ┌──────────────────────┐   ┌──────────────────┐
    │  viontin_framework│   │  viontin-cli          │   │  viontin-tui     │
    │  (core library)   │   │  45 commands           │   │  Terminal UI     │
    └──────────────────┘   └──────────────────────┘   └──────────────────┘
               │
               ▼
    ┌──────────────────┐
    │  viontin-macros   │
    │  #[domain] proc-  │
    │  macros           │
    └──────────────────┘
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
| `orm` | Core ORM: `QueryBuilder`, `Schema`, `Migration`, `Connection` traits, `Value`/`Row` types |
| `pg` | PostgreSQL driver (stub) |
| `mysql` | MySQL driver (stub) |
| `sqlite` | SQLite driver (stub) |

Each driver implements the core traits, allowing applications to switch databases by changing a single configuration value. No active-record Model layer — use the query builder directly to avoid architecture lock-in.

---

## CLI Reference

The `viontin` CLI provides 45 commands across these categories:

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
| `viontin-alpha` | Feature demo — includes `tailwind`, `serde`, `serde_json` |

---

## Quick Start

```bash
git clone https://github.com/viontin/viontin.git
cd viontin
bash scripts/install.sh --release

viontin new my-app
cd my-app
viontin dev
```

## Known Issues

See [known-issues](dev-tools/known-issues) for a comprehensive list of bugs, limitations, and technical debt.

## License

Open source software.
