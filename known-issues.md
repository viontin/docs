# Known Issues

> **Status:** This document catalogs all known issues, bugs, and technical debt across the Viontin project. Updated as of 2026-05-24. Items marked **[RESOLVED]** have been fixed.

Issues are categorized by severity:

| Severity | Label | Meaning |
|----------|-------|---------|
| 🔴 **Critical** | Memory safety, security, data loss, or crashes | Must fix before any production use |
| 🟡 **Major** | Broken features, incorrect behavior, build failures | Should fix before release |
| 🟢 **Minor** | Code quality, documentation gaps, warnings | Nice to fix |
| 🔵 **Suggestion** | Improvements, enhancements | Future consideration |

---

## 🔴 Critical Issues

### ~~C-001: Memory leak in Controller JSON parsing~~ **[RESOLVED]**

Fixed 2026-05-24: Replaced `Box::leak()` with `owned_to_ref()` pattern using owned `Vec<(String, Value)>` + temporary `&str` references. No more heap leak per request.

---

**Location:** `repos/framework/crates/framework/src/controller/mod.rs:165`  
**Type:** Memory safety / DoS vector

The `json_to_values()` function uses `Box::leak(k.clone().into_boxed_str())` to convert `String` keys from JSON request bodies into `&'static str` references for the `insert()` API. Every POST/PUT/PATCH request with a JSON body permanently leaks heap memory proportional to the size of the JSON key names.

**Impact:** Long-running services processing JSON requests will exhaust available memory over time. This is a denial-of-service vector.

**Root cause:** The `Repository::create()` and `Model::create()` APIs accept `Vec<(&str, Value)>` where the key has lifetime `'static` but the keys come from runtime-parsed JSON.

**Fix:** Change the data APIs to accept owned `String` keys (`Vec<(String, Value)>`) instead of static `&str` references. Update `Repository::to_values()` to return owned strings.

---

### ~~C-002: 40+ `unwrap()` calls in production code~~ **[RESOLVED]**

Fixed 2026-05-24: Replaced 20+ unwraps in `route/mod.rs`, `domain/mod.rs`, `ws/mod.rs`, `config/mod.rs`, `csrf/mod.rs`, `collection/mod.rs` with `if let Ok` pattern, `.ok()?`, `.unwrap_or_default()`, and `.expect()` with clear messages. Remaining unwraps are in test code and server path matching (guarded by prior checks).

---

**Location:** Multiple files across framework, CLI, gems  
**Type:** Runtime safety

The codebase uses `.unwrap()` extensively in production paths. Every unwrap is a potential runtime panic if the `Option` is `None` or the `Result` is `Err`.

**Key locations:**

| File | Line(s) | Context |
|------|---------|---------|
| `boot/mod.rs` | 43 | `ws_server.run(addr).unwrap()` — boot will panic if bind fails |
| `domain/mod.rs` | 110, 126 | Domain registry lock unwraps — panic on poisoned mutex |
| `route/mod.rs` | 94, 145 | Route registry lock unwraps |
| `server/mod.rs` | 162 | `position(|s| s == "*").unwrap()` — may panic on malformed routes |
| `csrf/mod.rs` | 104 | Lock unwrap in CSRF token generation |
| `ws/mod.rs` | 48, 127 | Route insertion unwraps |
| `project.rs` | 102, 106 | CLI project commands |
| `viontin-gem-webview/src/lib.rs` | 106, 128 | Gem lifecycle hooks |

**Fix:** Replace all `unwrap()` with proper error propagation using `?` or `map_err()` with meaningful error messages.

---

### ~~C-003: Rate limiter `too_many_attempts()` consumes an attempt~~ **[RESOLVED]**

Fixed 2026-05-24: `too_many_attempts()` now uses read-only `hits()` instead of `attempt()`. No more side effect on check.

---

**Location:** `repos/framework/crates/framework/src/rate/mod.rs:86-88`  
**Type:** Logic bug

`RateLimiter::too_many_attempts()` delegates to `TokenBucketLimiter::too_many_attempts()`, which internally calls `self.attempt(key, max_attempts, 1)`. The `attempt()` method **increments the hit counter** atomically. This means merely *checking* the rate limit consumes one allowed attempt.

```rust
// Bug: this check consumes one attempt
if RateLimiter::too_many_attempts("login:admin", 5) {
    // Rate limited — but one attempt was already consumed by the check
}

// Correct pattern would be a read-only check
```

**Impact:** After 4 failed login attempts, the 5th check will show "rate limited" even though only 4 real attempts occurred.

**Fix:** Split `too_many_attempts()` into a read-only method that does not increment the counter, or fix the implementation to use a separate counter for check vs. hit.

---

### ~~C-004: Domain feature fails to compile~~ **[RESOLVED]**

Fixed 2026-05-24: Added `From<String>` and `From<&str>` for `FrameworkError`. Renamed DDD `Repository` to `DomainRepository` to avoid name conflict with data access `Repository`.

---

**Location:** `repos/framework/crates/framework/src/domain/mod.rs:173`  
**Type:** Build failure

When `cargo check --features domain` is run, the domain module fails with a type error. The `?` operator in the boundary checking code cannot convert `String` to `FrameworkError` because `String` does not implement `Into<FrameworkError>`.

```rust
// Causes compile error:
let violations = DomainBoundary::scan_imports(project_root, &domains)?;
//            ^ cannot convert `String` to `FrameworkError`
```

**Impact:** The entire `domain` feature is unusable. Users cannot use DDD building blocks or `viontin check --arch`.

**Fix:** Implement `From<String> for FrameworkError` or change the function signatures to use proper error types.

---

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

```rust
// Current code — runs forever:
for stream in listener.incoming() {
    match stream {
        Ok(s) => { thread::spawn(move || { handle_conn(s, &router, &routes); }); }
        Err(e) => { eprintln!("  [server] {}", e); }
    }
}
```

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
pub struct QueryScoped<'a, M: Entity, R: Repository<M> + 'a> {
    _marker: ::std::marker::PhantomData<&'a (M, R)>,
}

impl<'a, M: Entity, R: Repository<M>> QueryScoped<'a, M, R> {
    pub fn where_eq(self, _col: &str, _val: impl Into<Value>) -> Self { self }
    pub fn all(&self) -> Result<Vec<M>, String> { Ok(vec![]) }   // always empty
    pub fn first(&self) -> Result<Option<M>, String> { Ok(None) } // always None
    pub fn count(&self) -> Result<u64, String> { Ok(0) }           // always 0
}
```

No `where_eq` conditions are actually applied. `all()` always returns an empty vec. `first()` always returns `None`. `count()` always returns `0`.

**Impact:** Scoped queries via `repo.query().where_eq("active", true).all()` silently return empty results instead of filtering. This can cause data loss or incorrect application behavior without any error signal.

**Fix:** Implement a real scoped query mechanism that maps conditions into `QueryBuilder` calls and delegates to `Repository::all()`.

---

### C-010: XOR encryption is NOT cryptographically secure

**Location:** `repos/framework/crates/framework/src/encryption/mod.rs`  
**Type:** Security

The built-in `SimpleEncrypter` uses XOR with a salt byte. This is trivially reversible with known-plaintext attacks and provides no real security. The documentation warns against production use but provides no production alternative — no AES implementation exists anywhere in the codebase.

```rust
// Algorithm: plaintext → XOR with key → add salt → hex encode
// This is NOT encryption — it's obfuscation
```

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

`FrameworkError` itself is a single-variant enum that is essentially `String` in a wrapper:

```rust
pub enum FrameworkError {
    #[error("{0}")]
    Internal(String),
}
```

This provides no structured error information — no error codes, no HTTP status mapping, no recovery hints, no typed error variants.

**Impact:** Callers cannot programmatically distinguish between "not found", "validation error", "database connection lost", or "permission denied". All errors must be string-matched, which is fragile and non-idiomatic Rust.

**Fix:** Introduce proper error enums for each layer (e.g., `RepositoryError`, `ServiceError`) and add typed variants. Implement `From` conversions and `std::error::Error` for compatibility.

---

### ~~C-012: LogEntry timestamp is never populated~~ **[RESOLVED]**

Fixed 2026-05-24: Added `timestamp()` helper producing ISO 8601 format. All `Logger::info()`, `error()`, `warning()`, `debug()` now populate the field.

---

**Location:** `repos/framework/crates/framework/src/log/mod.rs:70-73`  
**Type:** Data loss

Convenience methods `Logger::info()`, `Logger::error()`, `Logger::warning()`, and `Logger::debug()` create `LogEntry` instances with `timestamp: String::new()` (empty string). The timestamp is never populated with the current time.

```rust
pub fn info(&self, message: impl Into<String>) {
    self.log(LogEntry {
        level: Level::Info,
        message: message.into(),
        channel: "app".into(),
        context: Vec::new(),
        timestamp: String::new(),  // <-- always empty
    });
}
```

**Impact:** All log entries have empty timestamps, making log analysis, filtering, and debugging impossible.

**Fix:** Populate `timestamp` with ISO 8601 formatted current time in each convenience method, or populate it inside `Logger::log()`.

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

1. `env/mod.rs:45` — `unsafe { std::env::set_var(&key, &value); }` — calling `set_var` is unsafe in multi-threaded contexts because it can race with reads. No mutex, no synchronization comment.
2. `cli/output.rs:348` — Windows console mode setting via `SetConsoleMode`. No safety comment explaining invariants.

**Impact:** Potential data races in multi-threaded environments and undefined behavior on Windows.

**Fix:** Add safety comments documenting what invariants are maintained. For `set_var`, consider moving to initialization-only (before threads spawn). Use `OnceLock` or similar synchronization.

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
- `non_snake_case` — naming convention violations
- `unused_assignments` — values assigned but never read

**Fix:** Run `cargo clippy --fix` and manually review remaining warnings.

---

### M-002: `#[allow(dead_code)]` annotations mask unused code

**Location:** `repos/framework/crates/cli/src/project.rs`, `orm/src/schema.rs`, `cli/src/commands/check.rs`, and more  
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
domain = ["viontin-framework/domain"]
orm = ["viontin-framework/orm"]
default = []
```

The `default = []` means installing `viontin` as a dependency gives the user a minimal framework with no ORM, no domain-driven design, and no async support. This is surprising — users adding `viontin` expect a batteries-included experience.

**Impact:** New users who follow "quick start" guides that use ORM features will get compilation errors because the `orm` feature is not enabled by default.

**Fix:** Change `default = ["orm"]` or document prominently that features must be enabled.

---

### M-005: Domain feature compile error

**Location:** `repos/framework/crates/framework/src/domain/mod.rs:173`  
**Type:** Build failure (duplicate of C-004, tracked here for priority)

Same as C-004. Listed again for visibility under Major issues as a build-affecting bug.

---

### M-006: No graceful shutdown (duplicate of C-006)

Listed under Critical. Tracked in both categories for emphasis.

---

### M-007: Example `.env` file committed

**Location:** `examples/viontin-alpha/.env`  
**Type:** Security / Bad practice

```env
APP_ENV=local
APP_DEBUG=true
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=viontin
DB_USERNAME=root
DB_PASSWORD=
```

While this example `.env` uses empty passwords, it sets a bad precedent. Real projects may copy this pattern and commit real secrets. Additionally, committed `.env` files cause issues with tools that auto-load `.env`.

**Fix:** Add `examples/viontin-alpha/.env` to `.gitignore` and rename to `.env.example`.

---

### M-008: No changelog or release process

**Location:** Entire repository  
**Type:** Process

There is no `CHANGELOG.md`, no release tags, no versioning strategy. All 14 crates are at `v0.1.0`. There is no deprecation policy, no migration guides, no `#[deprecated]` annotations on any APIs.

**Impact:** Users have no way to understand what changed between versions, what's deprecated, or how to migrate.

**Fix:** Create a `CHANGELOG.md`, establish a versioning strategy (semver), and start tagging releases.

---

### M-009: API surface not curated

**Location:** Every crate  
**Type:** API design

Most items are declared `pub` without consideration of what should be private. There are no `#[doc(hidden)]` annotations on internal implementation details. The visibility of the entire codebase needs review:

- Internal helper functions are publicly exported
- Implementation details of traits are public
- No internal `pub(crate)` visibility
- Re-exports in the facade crate are not curated

**Impact:** The public API surface is massive and unmaintainable. Breaking changes are inevitable because internal details are exposed.

**Fix:** Audit visibility modifiers. Use `pub(crate)` for internal items. Add `#[doc(hidden)]` for items that must be public for technical reasons but shouldn't be part of the API.

---

### M-010: `framework::errors` and `framework::error` coexist confusingly

**Location:** `repos/framework/crates/framework/src/error/`, `repos/framework/crates/framework/src/errors/`  
**Type:** Architecture / Dead code

The framework has two parallel error modules:
- `error/mod.rs` — defines `FrameworkError`, `Result<T>`, `SourceLocation`
- `errors/mod.rs` — defines `HttpError`, `ErrorReport`, error handler infrastructure

These two modules:
- Do not interoperate — `HttpError` doesn't use `FrameworkError`
- Overlap in concerns — both handle "errors"
- `errors/mod.rs` defines a registration system (`set_handler`) that is never called

The `errors/mod.rs` handler system appears to be partially implemented legacy code.

**Fix:** Either consolidate into a single error module, or clearly document the separation. Remove dead code in `errors/mod.rs`.

---

### M-011: Rocket-style route macro referenced but non-functional

**Location:** Documentation references  
**Type:** Documentation / Feature gap

Several documentation files reference route registration via a macro (`route::get(...)`, `route::post(...)`, etc.). These functions exist in `route/mod.rs` but only track metadata (handler name + source location) — they do NOT register executable handler functions. The actual handler registration requires separate calls:

```rust
// Documentation shows:
route::get("/users", "UserController::index", "routes/web.rs:10");

// But this only stores metadata! Actual handler requires:
route::register_handler(Method::Get, "/users", Arc::new(handler));
```

**Impact:** Developers following documentation examples will find that routes are tracked but never served.

**Fix:** Either integrate the handler functions with the metadata registration, or clarify the documentation that metadata-only registration requires a manual `extend_from_registry()` call.

---

### M-012: Multiple Cargo.lock files cause dependency drift

**Location:** All crate directories  
**Type:** Build / Reproducibility

At least 10 separate `Cargo.lock` files exist across the project:

```
repos/framework/crates/cli/Cargo.lock
repos/framework/crates/framework/Cargo.lock
repos/framework/crates/viontin/Cargo.lock
repos/framework/crates/macros/Cargo.lock (might not exist)
repos/framework/crates/tui/Cargo.lock
repos/orm/Cargo.lock
repos/gems/Cargo.lock
repos/viontest/Cargo.lock
examples/viontin-zero/Cargo.lock
examples/viontin-alpha/Cargo.lock
```

Different workspaces may resolve the same dependency to different versions, causing inconsistent behavior across crates.

**Fix:** Create a root workspace `Cargo.toml` that includes all crates, producing a single `Cargo.lock` for the entire project.

---

### M-013: Edition 2024 constrains MSRV

**Location:** All `Cargo.toml` files  
**Type:** Compatibility

All crates use `edition = "2024"`, which requires Rust 1.85+. This is very recent (stable February 2025). Users on:
- Ubuntu 24.04 LTS repositories (typically ship Rust 1.75 or so)
- Enterprise Linux (RHEL 9 — Rust 1.71)
- Older CI environments

...cannot compile the project without manual `rustup` updates.

**Fix:** Consider `edition = "2021"` for broader compatibility, or document the MSRV requirement prominently.

---

### M-014: No `serde` support in ORM by default

**Location:** `repos/orm/crates/viontin-orm/Cargo.toml`  
**Type:** Usability

```toml
[dependencies]
serde = { version = "1", features = ["derive"], optional = true }
```

`serde` is optional in the ORM crate but is needed for `Row` serialization, `Page<T>` serialization, and JSON responses. If serde is not enabled, the ORM's most common use cases (returning JSON from controllers) require manual serialization.

**Impact:** Users who add `viontin-orm` as a standalone dependency without enabling `serde` will find that `QueryBuilder::get()` returns `Row` types that can't be serialized to JSON.

**Fix:** Enable `serde` by default or add a dedicated serialization feature.

---

### M-015: `minimum_sql()` capability profile is incorrect

**Location:** `repos/orm/crates/viontin-orm/src/connection.rs:106-118`  
**Type:** Misleading API

```rust
pub fn minimal_sql() -> Self {
    DriverCapabilities { joins: false, ... }
}
```

The "minimal SQL" profile marks `joins` as unsupported. However, SQLite (which this profile is named for) does support `JOIN` operations. This misclassification prevents `viontin-orm` features from working correctly with SQLite.

**Fix:** Rename to clarify the intent, or set `joins: true` for SQLite compatibility.

---

### M-016: SyncQueue `later()` is synchronous

**Location:** `repos/framework/crates/framework/src/queue/mod.rs`  
**Type:** Logic bug

The `SyncQueue` driver ignores the delay parameter in `later()`:

```rust
impl Driver for SyncQueue {
    fn later(&self, _delay_secs: u64, job: Box<dyn Job>) -> Result<(), String> {
        self.push(job)  // executes immediately, ignoring delay
    }
}
```

**Impact:** Jobs scheduled with a delay execute immediately instead of after the specified delay.

**Fix:** Implement actual delayed execution (e.g., spawning a thread that sleeps), or document that `SyncQueue` does not support delayed jobs.

---

### M-017: No request ID or tracing correlation

**Location:** Missing feature  
**Type:** Observability

There is no request ID generation, no `tracing` spans, and no way to correlate log entries with specific requests. In a multi-threaded server, log entries from concurrent requests are interleaved with no way to tell which request produced which log.

**Impact:** Debugging production issues is extremely difficult without request correlation.

**Fix:** Integrate the `tracing` crate for span-based observability, generate request IDs in middleware, and attach them to log entries.

---

## 🟢 Minor Issues

### m-001: Missing doc comments on public APIs

**Location:** Multiple files — `collection/mod.rs`, `queue/mod.rs`, `storage/mod.rs`, `http/mod.rs`, and others  
**Type:** Documentation

Many public structs, enums, and methods lack `///` doc comments:

| Module | Missing docs for |
|--------|-----------------|
| `collection` | Several `pub fn` methods on `Collection<T>` |
| `queue` | `Driver` trait methods, `Job` trait |
| `storage` | `Driver` trait methods, `LocalStorage` |
| `http` | Various `Response` builder methods |
| `support` | `Hasher`, `Encrypter` trait docs |
| `validator` | `Outcome`, `Finding` fields |

**Impact:** Developers using IDE autocomplete or reading generated docs on docs.rs get no guidance on API usage.

**Fix:** Add doc comments to all public items.

---

### m-002: READMEs reference 404 GitHub URLs

**Location:** `docs/README.md`, `repos/framework/README.md`, `repos/orm/README.md`, `repos/gems/README.md`, `repos/viontest/README.md`  
**Type:** Documentation

All README files reference `https://github.com/viontin/docs` which returns a 404. References to "the Viontin website" and "the documentation site" point to non-existent URLs.

**Impact:** Developers trying to find more information encounter dead ends.

**Fix:** Update URLs to point to actual locations (or remove them if no documentation site exists yet).

---

### m-003: No root `.gitignore`

**Location:** Root directory  
**Type:** Repository hygiene

The root directory has no `.gitignore` file. Individual workspaces have their own `.gitignore` files, but artifacts created at the root level (build outputs, editor files, OS files) are not excluded.

**Impact:** Accidental commits of editor swap files, build artifacts, or `.env` at the root level.

**Fix:** Add a root `.gitignore` with standard Rust ignores and OS/editor patterns.

---

### m-004: No `rustfmt.toml` or formatting guarantee

**Location:** Entire repository  
**Type:** Code quality

There is no `.rustfmt.toml` configuration file. Code formatting varies across the codebase — some files use long lines (300+ characters), some have inconsistent indentation, and there is no CI enforcement.

**Impact:** Inconsistent code style makes code reviews harder and diffs noisier.

**Fix:** Add `rustfmt.toml` (e.g., `max_width = 120`, `tab_spaces = 4`), format the entire codebase with `cargo fmt`, and enforce in CI.

---

### m-005: Inconsistent naming conventions

**Location:** `repos/framework/crates/viontin/src/boot/mod.rs`  
**Type:** Code quality

The `Boot` builder had camelCase methods (`withProviders`, `withoutProvider`, etc.) that were recently renamed to snake_case (`with_providers`, `without_provider`). However, some code may still reference the old names, and not all naming is consistent across the framework.

Additionally, module names are inconsistent:
- Some use singular: `controller`, `service`, `entity`
- Some use plural: `events`, `errors`, `notifications`

**Impact:** Developer confusion and potential compilation errors when old API names are used.

**Fix:** Complete the naming audit and ensure consistency.

---

### m-006: License file exists but no third-party attribution

**Location:** `LICENSE-MIT`  
**Type:** Legal

The project has an MIT license file but no `THIRD_PARTY` or `NOTICE` file listing dependencies and their licenses. While MIT is a permissive license, some dependencies may have different terms (e.g., Apache-2.0, MPL-2.0) that require attribution.

**Impact:** Potential legal non-compliance when distributing the application.

**Fix:** Run `cargo-deny` or similar to generate a third-party license report.

---

### m-007: Bench module name collisions

**Location:** `repos/framework/crates/framework/src/server/mod.rs`  
**Type:** Code quality

The test module inside `ws/mod.rs` is named `tests`, which is standard. However, the `benchmark` function in `debug/mod.rs` creates a potential naming collision with the `bench` testing feature.

**Impact:** Minor — only affects documentation clarity.

---

### m-008: `dump` and `dd` functions return no value

**Location:** `repos/framework/crates/framework/src/debug/mod.rs`  
**Type:** Usability

Unlike Laravel's `dd()` which terminates execution and can be used inline, Viontin's `dd()` takes a reference and exits:

```rust
pub fn dd(value: &dyn fmt::Debug) -> ! {
    // ...
    std::process::exit(0);
}
```

This makes it unusable in expressions.

**Fix:** Consider making `dd()` generic over owned values and returning them for use in expressions, similar to Laravel's `dd()` behavior.

---

### m-009: No error page rendering

**Location:** `repos/framework/crates/framework/src/errors/mod.rs`  
**Type:** User experience

Error responses are sent as plain text:

```rust
Response::html("Not found")
```

There are no styled error pages for development mode (like Laravel's "Whoops!" pages or Symfony's debug toolbar). Production error pages are equally bare.

**Impact:** Development debugging is harder without rich error pages showing stack traces, request data, and environment state.

---

### m-010: Viontin examples compile with dead-code warnings

**Location:** `examples/viontin-alpha/`  
**Type:** Quality

The example app compiles with 11 warnings about unused structs and functions:

```
warning: struct `NewFollower` is never constructed
warning: struct `WelcomeEmail` is never constructed
warning: struct `UserRegistered` is never used, etc.
```

**Impact:** Sets a bad example for users who look at example code for patterns.

**Fix:** Either use the declared items in the example, or add `#[allow(dead_code)]` with justification.

---

## 🔵 Suggestions

### S-001: Add tracing/OpenTelemetry integration

**Severity:** Suggestion

Integrate the `tracing` crate for structured, async-aware diagnostics with span-based instrumentation. Consider OpenTelemetry for distributed tracing in microservices deployments.

### S-002: Add metrics with `metrics` crate

**Severity:** Suggestion

Add a `metrics` module with counters, histograms, and gauges for common operations:
- Request count, latency, status code distribution
- Database query count and latency
- Cache hit/miss ratios
- Queue depth and processing time

### S-003: Add property-based testing

**Severity:** Suggestion

Use `proptest` or `quickcheck` for property-based testing of core components:
- Round-trip serialization of HTTP messages
- SQL query builder invariants
- Cache consistency properties
- Event dispatcher ordering

### S-004: Add fuzz testing for HTTP parser

**Severity:** Suggestion

The hand-written HTTP/1.1 parser is a critical surface for security vulnerabilities. Add fuzz testing using `cargo-fuzz` or `afl` to test malformed request handling.

### S-005: Add Redis cache driver

**Severity:** Suggestion

Only `MemoryCache` and `FileCache` exist. Add `RedisCache` implementing `CacheDriver` for production caching.

### S-006: Add AES-256-GCM encryption

**Severity:** Suggestion

Replace or supplement `SimpleEncrypter` with a proper AES-256-GCM implementation using the `aes-gcm` crate for production-grade encryption.

### S-007: Add connection pooling to HTTP server

**Severity:** Suggestion

The current HTTP server creates a new thread per connection with no keep-alive support. Add HTTP/1.1 keep-alive with a connection pool for better performance.

### S-008: Default features should include `orm`

**Severity:** Suggestion

Consider `default = ["orm"]` in the meta-crate so users get a batteries-included experience while still being able to opt out with `default-features = false`.

---

## How to Read This Document

Issues are listed in descending severity order. Within each severity level, issues are roughly ordered by impact.

**Legend:**
- 🔴 = Must fix before any production use
- 🟡 = Should fix before release
- 🟢 = Nice to fix
- 🔵 = Future consideration

This document should be updated as issues are resolved. When an issue is fixed, mark it with **[RESOLVED]** and add the date and fix description.

---

## See Also

- [README](README) — Project overview
- [Getting Started](getting-started) — Quick start guide
- [Architecture](architecture) — System design
