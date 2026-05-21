# Philosophy

**Zero to One, Scale-up Easily** — from prototype to production fleet on one platform.

---

Viontin is built on a single conviction: **a framework should accelerate you at every stage** — from the first `fn main()` to a multi-team production system — without ever forcing a rewrite.

This document explains the principles that guide every design decision in the platform.

---

## 1. Zero to One: From Nothing to Running

The distance between an empty directory and a running application should be measured in seconds, not hours.

### One Dependency

```toml
[dependencies]
viontin = { path = "../../repos/framework/crates/viontin" }
```

A single crate import unlocks the entire platform: HTTP server, CLI, TUI, config, caching, auth, events, mail, queues, scheduling, validation, i18n, storage, debugging, architectural testing. No hunting for the right combination of half-dozen crates. No version mismatch. No "will these crates work together?"

### One Boot Call

```rust
fn main() {
    viontin::boot()
        .get("/", |_| Response::html("Hello, World!"))
        .serve("127.0.0.1:3000");
}
```

From zero to a running web server in one function chain. The `boot()` function initializes the DI container, config loader, CLI kernel, router, WebSocket router, and gem registry — all with sensible defaults. You pay for what you use, but nothing requires explicit setup.

### Convention Over Configuration

Every decision has a rational default:

- Config lives in `config/` — no schema required
- `.env` files are loaded automatically
- Templates go in `html/`
- Routes are registered via a global registry or builder
- Environment is auto-detected from `APP_ENV`
- Logging works out of the box to stdout

You override when you need to. You never configure just to start.

### Scaffolding Is Not an Afterthought

```bash
viontin new my-app       # complete project in one command
viontin make:controller  # controller with route registration
viontin make:model       # model with migration, relationships
viontin make:domain      # bounded context with port
```

The 33 CLI commands exist because **scaffolding is a first-class feature**, not a plugin. Every `make:*` command produces production-viable code, not stubs.

### Content Embedding

```rust
use viontin::{html, md, js, ts};

let page = html!("templates/index.html");   // compile-time
let docs = md!("docs/guide.md");             // compile-time
let script = js!("assets/app.js");           // compile-time
```

Assets are embedded at compile time — zero file I/O at runtime, zero configuration. The macros are simple `include_str!` wrappers today, extensible for minification or transpilation tomorrow.

---

## 2. Scale-up Easily: Growing Without Pain

Prototypes are easy. Production systems with five teams and a million users are hard. Viontin grows with you.

### Progressive Architecture (Level 0 → 3)

The platform defines four maturity levels. You start at Level 0 and move up only when you need to:

| Level | Name | What You Get |
|-------|------|-------------|
| 0 | Solo | One `main.rs`, one dependency, one boot call |
| 1 | Team | Scaffolding, modules, conventions, `make:*` commands |
| 2 | Domain | Bounded contexts, `domain!()` macro, `viontin check --arch` |
| 3 | Fleet | Service contracts, gem plugins, CI-enforced architecture rules |

You are never locked into a level. Going from Level 0 to Level 2 is extracting domains from existing code — not rewriting.

### Additive Constraints

Every architectural rule in Viontin is **opt-in**:

- No domains? No boundary checking. Code works the same.
- No architecture tests? No problem. Skip them.
- Need to bypass a domain boundary? `allows` can be updated.

```rust
// The escape hatch is explicit, not hacky:
pub const DEFINITION: Domain = Domain::new("billing")
    .allows(&["order", "payment", "notification"]); // explicitly allowed
```

Rules provide **visibility, not restriction**. When you flag a violation, you can consciously accept it or fix it. The framework trusts the developer.

### Facade Pattern: Swap Without Changing Code

Every major subsystem uses a driver-based facade:

```rust
// Development: in-memory cache
let cache = Cache::memory();

// Production: file-backed cache
let cache = Cache::file("storage/cache/");

// Testing: null cache (no-op)
let cache = Cache::null();
```

The API never changes. Only the constructor. The same pattern applies to sessions, storage, mail, queue, auth, and notifications. You write your application logic once and switch backends by changing the driver.

### Plugin System (Gems)

When the framework itself isn't enough, you extend it via Gems — WASM-based plugins that hook into the build lifecycle:

```rust
boot()
    .gem(TailwindGem)          // CSS at build time
    .gem(CustomEncryptionGem)  // AES via a gem
    .gem(ImageProcessingGem)   // transform at deploy time
```

Gems integrate at the `before_build` and `after_build` phases — they can process assets, generate code, validate configuration, or run external tools — all without forking the framework.

### Architectural Testing

As teams grow, naming conventions and dependency rules need enforcement:

```rust
let mut checker = ArchChecker::new();
checker.add(IsPascalCase);                          // struct naming
checker.add(DoesNotDependOn::new("old_repo_module")); // forbidden deps
checker.add(EndsWith::new("Controller"));             // naming convention

// Run in CI:
let result = checker.check_all(modules);
assert!(result.passed(), "Architecture violations found");
```

Rules are code. They live in your test suite. They run in CI. They prevent drift without manual code reviews.

### Multi-Driver ORM

```toml
# Switch from SQLite to PostgreSQL:
# viontin-orm-sqlite → viontin-orm-pg
# Change DATABASE_URL in .env
# Done.
```

The ORM follows the same principle as facades — core traits with driver-specific implementations. Models, migrations, schemas, and relations are defined once and work across all supported databases.

---

## 3. Platform Thinking: Beyond Web

Viontin is not a web framework. It is a **platform that includes a web framework**.

The same crate, the same boot sequence, the same DI container can produce:

```rust
// A web server
boot()
    .get("/", handler)
    .serve("127.0.0.1:3000");

// A CLI tool
boot()
    .command(DeployCommand)
    .command(RollbackCommand)
    .run(|_| {});

// A TUI application
boot()
    .command(InteractiveWizard)
    .run(|_| {});

// A batch processor
boot()
    .command(ProcessQueueCommand)
    .provider(QueueWorkerProvider)
    .run(|app| {
        loop { app.queue().process_next() }
    });
```

The architectural patterns — facades, service providers, CLI commands, configuration, logging — are shared across all four modes. A mail notification you write for the web also works in a CLI command. A queue job you define for the web can be processed by a separate worker binary compiled from the same codebase.

---

## 4. Pure Rust, No Magic

### No DSLs

```rust
// Domain declaration — not a DSL, just Rust:
pub const DEFINITION: Domain = Domain::new("billing")
    .allows(&["order", "payment"]);
```

The `domain!` macro is syntactic sugar, not a new language. Every structural concept maps directly to a Rust struct, trait, or function.

### No Proc Macros

Zero custom derive macros. Zero attribute macros. The framework uses:
- Regular Rust traits for extension
- `include_str!` for content embedding
- `TypeId` for the DI container
- Regex for architecture scanning

Proc macros add compilation time, obscure control flow, and create tooling friction. They are avoided unless there is no alternative.

### No Code Generation

Scaffolding creates files on disk. Architecture checking scans existing code. There is no "codegen" step between your source and the compiler. What you see is what the compiler sees.

---

## 5. Developer Experience First

### Debugging Is a First-Class Feature

```rust
dump!(user);         // prints: src/routes/users.rs:42 → User { id: 1, name: "Alice" }
dd!(request);        // same as dump, but exits immediately
```

`dump` and `dd` mirror the ergonomics of PHP's `var_dump` and Ruby's `puts` — a muscle-memory reflex that should be one keystroke away, not a `println!("{:?}", ...)` wrapped in `#[derive(Debug)]`.

```rust
let mut p = Profiler::new();
p.start("db-query");
let users = db.query("SELECT * FROM users");
p.end("db-query");
println!("{}", p.report());  // db-query: 12.34ms
```

### Zero Cargo at Runtime

The `viontin` CLI is a standalone binary — 33 commands, zero `cargo` invocations at runtime. No `cargo run`, no `cargo build` for `dev` mode. The CLI wraps cargo transparently for build commands, but the development server, scaffolding, inspection, and architecture checking all run without the Rust toolchain.

### Errors With Solutions

```rust
struct ErrorReport {
    title: String,
    detail: String,
    solution: Option<String>,    // "Try running: viontin make:domain billing"
    file: SourceLocation,
    hints: Vec<String>,
}
```

Framework errors include actionable solutions. When a route conflicts, the error shows both source locations. When a domain boundary is violated, the output shows the exact file, line, and import. The goal is to turn every error into a teaching moment.

---

## 6. Zero External HTTP Dependencies

The HTTP server, WebSocket handler, request parser, and response writer are all hand-written on top of `TcpListener` and `TcpStream`:

```
Server          → TcpListener (std::net)
Request parse   → manual HTTP/1.1 parser
WebSocket       → hand-rolled SHA-1 + Base64
Router          → trie-free, linear scan with path params
```

**Why?** Full control. Zero dependency bloat. No `hyper`, no `actix`, no `tokio`. The framework compiles quickly, has a small dependency tree, and every byte of network code is owned and understood by the maintainers.

This is a deliberate trade-off. When the platform needs HTTP/2, connection pooling, or async I/O, those capabilities will be added to the hand-written server — not pulled in from an external runtime.

---

## 7. Rust Edition 2024

Viontin targets Rust edition 2024 from day one. This means:

- Modern `impl Trait` syntax
- `gen` blocks when stable
- Enhanced RPITIT (Return Position Impl Trait In Traits)
- The latest borrow checker (Polonius-ready)

The framework is designed to track Rust's evolution. Edition 2024 is not a goal — it is a starting point.

---

## 8. Comparison Principles

| Principle | Viontin | Other Frameworks |
|-----------|---------|-----------------|
| **Dependency count** | Single meta-crate | 5–15 crates for similar features |
| **HTTP networking** | Hand-written | hyper / actix / tokio |
| **Proc macros** | Zero | Common in Rust web frameworks |
| **Async model** | Synchronous (threads) | Async (tokio/async-std) |
| **Maturity levels** | Explicit (0→3) | Implicit or absent |
| **Arch enforcement** | Built-in (arch testing + domains) | External tools only |
| **CLI** | 33 commands, standalone binary | Usually cargo subcommand |
| **TUI** | First-class toolkit | None or external |
| **Plugin system** | WASM-based Gems | Usually middleware only |

Viontin is not trying to compete on async throughput or zero-cost abstractions. It competes on **developer experience, architectural clarity, and progressive complexity** — the things that matter when building real products with real teams.

---

## 9. The Name

**Viontin** is derived from "zero to one" — the journey from nothing to something, and then scaling that something without breaking it.

The tagline captures both halves:

> **Zero to One** — Get from idea to working prototype as fast as possible.  
> **Scale-up Easily** — Take that prototype to production without rewriting it.

One platform. One dependency. Every stage of growth.
