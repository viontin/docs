# Known Issues

> **Status:** This document catalogs all known issues, bugs, and technical debt across the Viontin project. Updated as of 2026-05-24. Resolved items have been removed from this list.

Issues are categorized by severity:

| Severity | Label | Meaning |
|----------|-------|---------|
| 🔴 **Critical** | Memory safety, security, data loss, or crashes | Must fix before any production use |
| 🟡 **Major** | Broken features, incorrect behavior, build failures | Should fix before release |
| 🟢 **Minor** | Code quality, documentation gaps, warnings | Nice to fix |
| 🔵 **Suggestion** | Improvements, enhancements | Future consideration |

---

## 🔴 Critical Issues

### C-005: Zero test coverage

**Location:** Entire codebase  
**Type:** Quality assurance

The entire project contains only 9 `#[test]` functions, all located in `repos/framework/crates/framework/src/ws/mod.rs` (WebSocket frame encoding/decoding tests). All other modules — Entity, Model, Repository, Service, Controller, Auth, Cache, Config, Encryption, Events, Queue, Rate Limiter, etc. — have **zero tests**.

Additionally:
- No integration tests (`tests/` directories) exist anywhere
- No property-based testing
- No documentation tests (`/// ```rust` examples that compile)
- No CI pipeline runs any test command

**Impact:** Any refactoring or change risks breaking existing functionality with no safety net.

**Fix:** Add tests incrementally, starting with the most critical modules: ORM QueryBuilder, Repository traits, Auth, and boot sequence.

---

### C-006: No graceful shutdown

**Location:** `repos/framework/crates/framework/src/server/mod.rs:178-195`  
**Type:** Operational

The HTTP server runs in an infinite `for stream in listener.incoming()` loop with no signal handling (SIGTERM, SIGINT). When the process receives a termination signal, in-flight requests are immediately dropped and connections are aborted. There is no:
- Signal handler for graceful shutdown
- Drain mechanism for in-flight requests
- Connection close handshake
- Timeout for pending requests

**Impact:** Production deployments (Kubernetes, Docker, systemd) will abort active connections during rolling updates or scale-down events.

**Fix:** Implement signal handling with `ctrlc` crate or `tokio::signal`, add a shutdown flag, and drain in-flight requests before exiting.

---

### C-007: No health check endpoints

**Location:** Missing feature  
**Type:** Operational

There is no built-in `/healthz` (liveness) or `/readyz` (readiness) endpoint. Load balancers, Kubernetes probes, and orchestration tools have no way to determine if the application is running or ready to serve traffic.

**Impact:** Cannot be deployed in container orchestration environments (Kubernetes, Nomad, ECS).

**Fix:** Add default health check routes to the Router that are always registered, returning 200 with basic application status.

---

### C-008: All ORM driver crates are empty stubs

**Location:** `repos/orm/crates/viontin-orm-pg/`, `viontin-orm-mysql/`, `viontin-orm-sqlite/`  
**Type:** Non-functional feature

All three database driver crates are placeholders. Every method returns `Err("Not implemented".to_string())`:

```rust
impl Connection for PgConnection {
    fn query(&self, _sql: &str, _params: &[Value]) -> Result<Vec<Row>, String> {
        Err("Not implemented".to_string())
    }
}
```

No actual database drivers exist for PostgreSQL, MySQL, or SQLite. The drivers have no dependencies on `sqlx` or `rusqlite`.

**Impact:** The entire ORM is non-functional. No database queries can be executed.

**Fix:** Implement actual database connectivity. For SQLite, add `rusqlite` dependency and implement the `Connection` trait. For PG/MySQL, add `sqlx` and implement async-aware connection pools.

---

### C-009: `QueryScoped` is a no-op

**Location:** `repos/framework/crates/framework/src/repository/mod.rs:131-136`  
**Type:** Non-functional feature

The `QueryScoped` builder type has method implementations that return hardcoded defaults:

```rust
pub fn where_eq(self, _col: &str, _val: impl Into<Value>) -> Self { self }
pub fn all(&self) -> Result<Vec<M>, String> { Ok(vec![]) }   // always empty
pub fn first(&self) -> Result<Option<M>, String> { Ok(None) } // always None
pub fn count(&self) -> Result<u64, String> { Ok(0) }           // always 0
```

**Impact:** Scoped queries via `repo.query().where_eq("active", true).all()` silently return empty results instead of filtering. This can cause data loss or incorrect application behavior without any error signal.

**Fix:** Implement a real scoped query mechanism that maps conditions into `QueryBuilder` calls and delegates to `Repository::all()`.

---

### C-010: XOR encryption is NOT cryptographically secure

**Location:** `repos/framework/crates/framework/src/encryption/mod.rs`  
**Type:** Security

The built-in `SimpleEncrypter` uses XOR with a salt byte. This is trivially reversible with known-plaintext attacks and provides no real security. The documentation warns against production use but provides no production alternative — no AES implementation exists anywhere in the codebase.

**Impact:** Any developer using `SimpleEncrypter` for real data protection has a false sense of security.

**Fix:** Either implement proper AES-256-GCM encryption as a built-in, or remove the feature entirely and provide clear guidance on using external encryption crates.

---

### C-011: All errors are stringly-typed

**Location:** All `Repository`, `Service`, `Controller`, `Model`, `Entity` traits  
**Type:** Architecture / Error handling

Every core trait returns `Result<_, String>` instead of a proper error type:

```rust
pub trait Repository<M: Entity> {
    fn all(&self) -> Result<Vec<M>, String>;
    fn save(&self, entity: &mut M) -> Result<M, String>;
    // ...
}

pub trait Service<M: Entity> {
    fn all(&self) -> Result<Vec<M>, String>;
    // ...
}
```

`FrameworkError` itself is a single-variant enum:

```rust
pub enum FrameworkError {
    #[error("{0}")]
    Internal(String),
}
```

**Impact:** Callers cannot programmatically distinguish between "not found", "validation error", "database connection lost", or "permission denied". All errors must be string-matched, which is fragile and non-idiomatic Rust.

**Fix:** Introduce proper error enums for each layer (e.g., `RepositoryError`, `ServiceError`) and add typed variants. Implement `From` conversions and `std::error::Error` for compatibility.

---

### C-013: No CI/CD pipeline

**Location:** Entire repository  
**Type:** Quality assurance

There is no CI/CD configuration of any kind:
- No GitHub Actions workflows
- No GitLab CI configuration
- No pre-commit hooks
- No automated test runs
- No automated linting or formatting checks

Additionally, there is no root-level `Cargo.toml` workspace. The project has 10 separate `Cargo.lock` files across independent workspaces, creating dependency version drift.

**Impact:** Every commit is unverified. Breaking changes, compilation errors, and test failures can be committed without detection.

**Fix:** Create a workspace-level `Cargo.toml`, add CI configuration (GitHub Actions recommended), and set up basic checks: `cargo check`, `cargo test`, `cargo clippy`, `cargo fmt --check`.

---

### C-014: No request timeout on TCP server

**Location:** `repos/framework/crates/framework/src/server/mod.rs`  
**Type:** Operational / Resource leak

The synchronous TCP server has no request timeout. A client can open a TCP connection, send the HTTP request line very slowly (e.g., 1 byte per minute), and hold the connection (and its thread) indefinitely. This is a resource exhaustion vector.

The async server (`server_async.rs`) also lacks `tokio::time::timeout`.

**Impact:** A single malicious or misconfigured client can exhaust the server's thread pool, causing denial of service.

**Fix:** Add read timeouts using `TcpStream::set_read_timeout()` for the sync server and `tokio::time::timeout` for the async server.

---

### C-015: Unsafe blocks without safety invariants

**Location:** `repos/framework/crates/framework/src/env/mod.rs:45`, `cli/output.rs:348`  
**Type:** Memory safety

Two `unsafe` blocks exist without documentation explaining why they are safe:

1. `env/mod.rs:45` — `unsafe { std::env::set_var(&key, &value); }` — calling `set_var` is unsafe in multi-threaded contexts. No mutex, no synchronization comment.
2. `cli/output.rs:348` — Windows console mode setting. No safety comment.

**Impact:** Potential data races in multi-threaded environments.

**Fix:** Add safety comments documenting what invariants are maintained. For `set_var`, consider moving to initialization-only (before threads spawn). Use `OnceLock` or similar synchronization.

---

### C-016: No panic recovery middleware

**Location:** Missing feature  
**Type:** Operational

If any request handler panics (e.g., `unwrap()`, `panic!()`, index out of bounds), the panic propagates to the thread level and `thread::spawn` catches it silently — but the entire connection handling thread dies without returning any response. The client hangs until timeout.

```rust
// thread::spawn in server/mod.rs — panic is silently caught
thread::spawn(move || {
    if let Err(e) = handle_conn(s, &router, &routes) {
        eprintln!("  [ws] {}", e);
    }
    // If handle_conn panics, the panic is caught by the thread
    // and the client connection is never closed
});
```

**Impact:** A panic in any handler leaves the client connection open indefinitely (until OS timeout). In the sync server, this leaks a thread per panic.

**Fix:** Add `std::panic::catch_unwind` around handler execution in the server's connection loop. Return a 500 Internal Server Error response if a panic is caught.

---

### C-017: No CORS middleware

**Location:** Missing feature  
**Type:** Operational

There is no built-in middleware for Cross-Origin Resource Sharing. Any application that serves a frontend from a different origin must build its own CORS middleware from scratch.

**Impact:** All web applications with separate frontend origins require manual CORS header handling. This is a common source of bugs and security misconfigurations.

**Fix:** Add a `CorsMiddleware` with configurable allowed origins, methods, and headers. Integrate with the existing `Middleware` trait.

---

### C-018: No rate limit middleware

**Location:** Missing feature  
**Type:** Operational

The `RateLimiter` facade exists as a standalone utility, but there is no `RateLimitMiddleware` that can be plugged into the middleware chain for route-level or global rate limiting.

**Impact:** Applications must manually check rate limits in every handler. There is no way to declaratively apply rate limiting to routes.

**Fix:** Create a `RateLimitMiddleware` that implements the `Middleware` trait, accepting a `RateLimiter` instance and a key extractor function.

---

### C-019: No static file serving middleware

**Location:** Missing feature  
**Type:** Missing functionality

The `Router` has no `static_files()` method. Serving static assets (CSS, JS, images) requires manually writing a handler that reads from disk for every request.

**Impact:** Every application must implement its own static file handler. This is error-prone and misses optimization opportunities (caching headers, `If-Modified-Since`, directory traversal protection).

**Fix:** Add a `StaticFiles` middleware or `Router::static_files()` method that serves files from a directory with proper caching headers and security checks.

---

### C-020: No built-in HTTP client

**Location:** Missing feature  
**Type:** Missing functionality

Viontin has no built-in HTTP client. Users must add `reqwest` or `ureq` as an external dependency for:
- Service-to-service communication in microservices
- Webhook callbacks
- External API integration
- The `RemoteServiceAdapter` (which currently returns `Err("not implemented")`)

**Impact:** The microservices pattern (`ServiceContract` + `RemoteServiceAdapter`) is non-functional without an HTTP client. Any integration with external services requires an additional dependency.

**Fix:** Add a minimal HTTP client (wrapping `ureq` or `minreq`) as an optional dependency behind a feature flag. Implement `RemoteServiceAdapter::handle()` using it.

---

### C-021: No background job queue

**Location:** `repos/framework/crates/framework/src/queue/mod.rs`  
**Type:** Non-functional feature

Only `SyncQueue` exists, which executes jobs synchronously in the current thread:

```rust
impl Driver for SyncQueue {
    fn push(&self, job: Box<dyn Job>) -> Result<(), String> {
        job.handle()  // executes immediately, blocks caller
    }
    fn later(&self, _delay_secs: u64, job: Box<dyn Job>) -> Result<(), String> {
        self.push(job)  // ignores delay
    }
}
```

There is no persisted queue (database, file, or Redis-backed). Jobs cannot survive process restarts. Scheduled/delayed jobs execute immediately.

**Impact:** Background job processing is non-functional for production use. The scheduler is useless without a queue that can persist and delay jobs.

**Fix:** Implement a `DatabaseQueue` that stores jobs in SQLite (or other database via ORM) and processes them asynchronously in a background thread. Add proper delay support.

---

### C-022: No request tracing or correlation ID

**Location:** Missing feature  
**Type:** Observability

There is no request ID generation, no `tracing` spans, and no way to correlate log entries with specific requests. In a multi-threaded server, log entries from concurrent requests are interleaved with no way to tell which request produced which log.

**Impact:** Debugging production issues is extremely difficult without request correlation.

**Fix:** Integrate the `tracing` crate for span-based observability, generate request IDs in middleware, and attach them to log entries.

---

### C-023: No CORS, no panic recovery, no static files — missing middleware trio

These three are listed above as individual issues. Combined, they represent a significant gap in production readiness — every application needs them and must build them from scratch.

---

## 🟡 Major Issues

### M-001: 19+ clippy warnings across codebase

**Location:** All crates  
**Type:** Code quality

Running `cargo clippy` reveals warnings including:
- `duplicated_attributes` — duplicate `#[cfg]` annotations
- `multiple_bound_locations` — trait bounds in multiple places
- `type_complexity` — over-complex nested types
- `neg_cmp_op_on_partial_ord` — inverted comparison on floating-point
- `unused_imports` — dead imports
- `unused_mut` — unnecessary mutability
- `unused_assignments` — values assigned but never read

**Fix:** Run `cargo clippy --fix` and manually review remaining warnings.

---

### M-002: `#[allow(dead_code)]` annotations mask unused code

**Location:** `project.rs`, `schema.rs`, `check.rs`, and more  
**Type:** Code quality

At least 9 `#[allow(dead_code)]` annotations exist across the codebase. Each masks genuinely unused code (fields, functions, types) that should either be removed, made public, or prefixed with underscore.

**Examples:**
- `Blueprint::create` field — never read
- `Migrator::connection` field — never read
- `exec_cargo_allow_fail` function — never called

**Fix:** Audit each `#[allow(dead_code)]` annotation. Remove unused code or add justified usages.

---

### M-003: Global singletons prevent testability

**Location:** Multiple modules  
**Type:** Architecture

The framework relies on global `OnceLock` / `Mutex` singletons throughout:

| Singleton | Module | Purpose |
|-----------|--------|---------|
| `REGISTRY` | `domain/mod.rs` | Domain definitions |
| `GLOBAL` | `rate/mod.rs` | Rate limiter state |
| `GLOBAL_LOGGER` | `log/mod.rs` | Logger |
| `GLOBAL` | `config/mod.rs` | Config repository |
| `ROUTE_HANDLERS` | `route/mod.rs` | Route handler storage |
| `HANDLER` | `errors/mod.rs` | Error handler |
| `GLOBAL` | `gem/mod.rs` | Gem registry |

These singletons:
- Cannot be reset between tests, causing test pollution
- Create hidden dependencies that aren't visible in function signatures
- Make parallel test execution impossible
- Prevent multiple independent instances of the framework

**Fix:** Migrate from global singletons to dependency injection using the `Application` container. Provide test helpers for creating isolated instances.

---

### M-004: Feature flags all off by default

**Location:** `repos/framework/crates/viontin/Cargo.toml`  
**Type:** User experience

```toml
[features]
default = []
```

The `default = []` means installing `viontin` as a dependency gives the user a minimal framework with no ORM, no domain-driven design, and no async support. Users adding `viontin` expect a batteries-included experience.

**Impact:** New users who follow "quick start" guides that use ORM features will get compilation errors because the `orm` feature is not enabled by default.

**Fix:** Change `default = ["orm"]` or document prominently that features must be enabled.

---

### M-005: `framework::errors` and `framework::error` coexist confusingly

**Location:** `repos/framework/crates/framework/src/error/`, `repos/framework/crates/framework/src/errors/`  
**Type:** Architecture / Dead code

The framework has two parallel error modules:
- `error/mod.rs` — defines `FrameworkError`, `Result<T>`, `SourceLocation`
- `errors/mod.rs` — defines `HttpError`, `ErrorReport`, error handler infrastructure

These two modules:
- Do not interoperate — `HttpError` doesn't use `FrameworkError`
- Overlap in concerns — both handle "errors"
- `errors/mod.rs` defines a registration system (`set_handler`) that is never called

**Fix:** Either consolidate into a single error module, or clearly document the separation. Remove dead code in `errors/mod.rs`.

---

### M-006: Multiple Cargo.lock files cause dependency drift

**Location:** All crate directories  
**Type:** Build / Reproducibility

At least 10 separate `Cargo.lock` files exist across the project. Different workspaces may resolve the same dependency to different versions.

**Fix:** Create a root workspace `Cargo.toml` that includes all crates, producing a single `Cargo.lock`.

---

### M-007: Edition 2024 constrains MSRV

**Location:** All `Cargo.toml` files  
**Type:** Compatibility

All crates use `edition = "2024"`, which requires Rust 1.85+. This is very recent (stable February 2025). Users on:
- Ubuntu 24.04 LTS repositories (typically ship Rust 1.75 or so)
- Enterprise Linux (RHEL 9 — Rust 1.71)
- Older CI environments

...cannot compile the project without manual `rustup` updates.

**Fix:** Consider `edition = "2021"` for broader compatibility, or document the MSRV requirement prominently.

---

### M-008: Rocket-style route macro referenced but non-functional

**Location:** Documentation references  
**Type:** Documentation / Feature gap

The `route::get(...)`, `route::post(...)` functions track only metadata (handler name + source location) — they do NOT register executable handler functions. The actual handler registration requires separate `route::register_handler()` calls.

**Impact:** Developers following documentation examples will find that routes are tracked but never served.

**Fix:** Either integrate the handler functions with the metadata registration, or clarify the documentation that metadata-only registration requires a manual `extend_from_registry()` call.

---

### M-009: Serde optional in viontin-orm but needed for JSON responses

**Location:** `repos/orm/crates/viontin-orm/Cargo.toml`  
**Type:** Usability

`serde` is optional in the ORM crate but is needed for `Row` serialization and `Page<T>` serialization. If serde is not enabled, common use cases (returning query results as JSON from controllers) require manual serialization.

**Fix:** Enable `serde` by default or add a dedicated serialization feature.

---

### M-010: SyncQueue `later()` ignores delay

**Location:** `repos/framework/crates/framework/src/queue/mod.rs`  
**Type:** Logic bug

The `SyncQueue` driver ignores the delay parameter in `later()`. Jobs scheduled with a delay execute immediately.

**Fix:** Either implement actual delayed execution or document that `SyncQueue` does not support delayed jobs.

---

## 🟢 Minor Issues

### m-001: Missing doc comments on public APIs

**Location:** Multiple files — `collection/mod.rs`, `queue/mod.rs`, `storage/mod.rs`, `http/mod.rs`, and others  
**Type:** Documentation

Many public structs, enums, and methods lack `///` doc comments.

**Fix:** Add doc comments to all public items.

---

### m-002: READMEs reference 404 GitHub URLs

**Location:** `docs/README.md`, all `repos/*/README.md`  
**Type:** Documentation

README files reference `https://github.com/viontin/docs` which returns 404.

**Fix:** Update URLs to point to actual locations.

---

### m-003: No root `.gitignore`

**Location:** Root directory  
**Type:** Repository hygiene

**Fix:** Add a root `.gitignore` with standard Rust ignores.

---

### m-004: No `rustfmt.toml` or formatting guarantee

**Location:** Entire repository  
**Type:** Code quality

**Fix:** Add `rustfmt.toml`, format the entire codebase, and enforce in CI.

---

### m-005: Inconsistent naming conventions

**Location:** Various files  
**Type:** Code quality

Some modules use singular names (`controller`, `service`) while others use plural (`events`, `errors`).

**Fix:** Audit and normalize naming.

---

### m-006: License file exists but no third-party attribution

**Location:** `LICENSE-MIT`  
**Type:** Legal

**Fix:** Run `cargo-deny` to generate a third-party license report.

---

### m-007: No error page rendering

**Location:** `repos/framework/crates/framework/src/errors/mod.rs`  
**Type:** User experience

Error responses are plain text with no styled error pages for development mode.

**Fix:** Add styled error pages for development (stack trace, request data) and simple pages for production.

---

### m-008: Examples compile with dead-code warnings

**Location:** `examples/viontin-alpha/`  
**Type:** Quality

The example app compiles with 11 warnings about unused structs and functions.

**Fix:** Either use the declared items or add `#[allow(dead_code)]` with justification.

---

## 🔵 Suggestions

### S-001: Add tracing/OpenTelemetry integration

Integrate the `tracing` crate for structured, async-aware diagnostics with span-based instrumentation.

### S-002: Add metrics with `metrics` crate

Add counters, histograms, and gauges for request count, latency, database query time, cache hit/miss ratios, and queue depth.

### S-003: Add property-based testing

Use `proptest` or `quickcheck` for property-based testing of core components.

### S-004: Add fuzz testing for HTTP parser

The hand-written HTTP/1.1 parser is a critical surface for security vulnerabilities. Add fuzz testing using `cargo-fuzz`.

### S-005: Add Redis cache driver

Only `MemoryCache` and `FileCache` exist. Add `RedisCache` implementing `CacheDriver`.

### S-006: Add AES-256-GCM encryption

Replace or supplement `SimpleEncrypter` with a proper AES-256-GCM implementation using the `aes-gcm` crate.

### S-007: Add connection pooling to HTTP server

Add HTTP/1.1 keep-alive with a connection pool for better performance.

### S-008: Default features should include `orm`

Consider `default = ["orm"]` in the meta-crate.

### S-009: Add HTTP/2 support

The hand-written HTTP/1.1 server may benefit from HTTP/2 support for performance.

---

## How to Read This Document

Issues are listed in descending severity order. Within each severity level, issues are roughly ordered by impact.

**Legend:**
- 🔴 = Must fix before any production use
- 🟡 = Should fix before release
- 🟢 = Nice to fix
- 🔵 = Future consideration

---

## See Also

- [README](README) — Project overview
- [Getting Started](getting-started) — Quick start guide
- [Architecture](architecture) — System design
