# Known Issues

> **Status:** This document catalogs all known issues, bugs, and technical debt across the Viontin project. Updated as of 2026-05-24. Resolved items have been removed.

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

Only 9 `#[test]` functions exist — all in WebSocket frame encoding. No integration tests, no property-based tests, no CI test runner.

**Fix:** Add tests incrementally starting with ORM QueryBuilder, Repository, Auth, and boot sequence.

---

### C-011: All errors are stringly-typed

Every trait returns `Result<_, String>`. `FrameworkError` is a single `Internal(String)` variant. Callers cannot distinguish between "not found", "validation error", "db connection lost", or "permission denied".

**Fix:** Introduce typed error enums per layer (`RepositoryError`, `ServiceError`, `HttpError`) with `From` conversions.

---

### C-013: No CI/CD pipeline

No GitHub Actions, no automated tests, no linting, no formatting checks. 10 separate `Cargo.lock` files across independent workspaces.

**Fix:** Create workspace `Cargo.toml`, add GitHub Actions with `cargo check`, `cargo test`, `clippy`, `fmt --check`.

---

### C-015: Unsafe blocks without safety invariants

Two `unsafe` blocks (`env/mod.rs:45`, `cli/output.rs:348`) lack documentation explaining why they are safe.

**Fix:** Add safety comments. For `set_var`, move to initialization-only before threads spawn.

---

### C-021: No background job queue

Only `SyncQueue` exists — jobs execute synchronously in the current thread. No persisted queue, no async processing.

**Fix:** Implement `DatabaseQueue` using SQLite, with a background worker thread.

---



### D-002: Error messages are not actionable

When an error occurs, the framework returns generic messages (`"Internal error"`, `"Not found"`) with no indication of:
- Where the error originated (file:line)
- What the developer should do to fix it
- The SQL query that failed (for DB errors)
- The request context

**Fix:** Implement `ErrorReport` with `solution: Option<String>`, `file: SourceLocation`, and `hints: Vec<String>`. Wire into `FrameworkError` and HTTP error responses.

---

## 🟡 Major Issues

### M-002: `#[allow(dead_code)]` annotations mask unused code

At least 9 annotations across `project.rs`, `schema.rs`, `check.rs`. Examples: `Blueprint::create`, `Migrator::connection`, `exec_cargo_allow_fail`.

**Fix:** Audit each annotation. Remove unused code or add justified usages.

---

### M-003: Global singletons prevent testability

Framework relies on `OnceLock`/`Mutex` singletons (`REGISTRY`, `GLOBAL_LOGGER`, `GLOBAL_CONFIG`, `ROUTE_HANDLERS`, etc.) that cannot be reset between tests.

**Fix:** Migrate to DI via `Application` container. Provide test helpers for isolated instances.

---

### M-005: `framework::errors` and `framework::error` coexist confusingly

Two parallel error modules that don't interoperate. `errors/mod.rs` defines a handler registration system never called.

**Fix:** Consolidate into a single error module or document the separation. Remove dead code.

---

### M-006: Multiple Cargo.lock files cause dependency drift

10+ separate lockfiles across independent workspaces can resolve same dependency to different versions.

**Fix:** Create a root workspace `Cargo.toml` with a single `Cargo.lock`.

---

### M-007: Edition 2024 constrains MSRV

Requires Rust 1.85+ (stable February 2025). LTS Linux distributions may ship older Rust versions.

**Fix:** Consider `edition = "2021"` or document MSRV prominently.

---

### M-009: `serde` optional in viontin-orm but needed for JSON responses

If `serde` is not enabled, common use cases (returning query results as JSON) require manual serialization.

**Fix:** Enable `serde` by default or document `features = ["serde"]`.

---

### D-003: `make:*` scaffolding does not update `mod.rs`

Each `make:controller User` creates `src/controllers/user.rs` but does not add `pub mod user;` to `src/controllers/mod.rs`. Developer must do it manually.

**Fix:** After writing the scaffold file, check if the parent `mod.rs` exists. If not, create it. If it exists and doesn't have the entry, append `pub mod snake;`.

---

### D-004: `make:migration` command missing

No CLI command to scaffold migration files with timestamps. Developers must write raw SQL and create files manually.

**Fix:** Add `make:migration create_users_table` that generates a timestamped `YYYYMMDDHHMMSS_create_users_table.rs` file in `src/migrations/` with up/down template.

---

### D-005: No query logging / profiling

Developers cannot see what SQL queries are executed, how long they take, or how many queries per request. Database optimization is blind.

**Fix:** Add a `QueryLogMiddleware` or `DB::listen()` callback that logs queries + duration at `debug` level. Also expose via a `Profiler`-style report.

---

### D-008: Feedback loop — no hot reload

`viontin dev` does not watch source files. Every code change requires manual `Ctrl+C` + `cargo run`. This breaks flow state and compounds Rust's compile time.

**Fix:** Integrate `notify` (already in CLI deps) to watch `src/` for changes and auto-restart the process. Same pattern as `cargo-watch`.

---

### D-009: Test setup is too heavy

Writing tests requires spinning up a full database, mocking multiple dependencies, and writing boilerplate. Most developers skip testing because the setup cost outweighs the perceived benefit. No `TestClient` for HTTP testing, no in-memory SQLite for repository testing, no factory for test data.

**Fix:** Provide `TestClient` that routes requests through the framework without a TCP listener. Provide `MemoryRepository` default implementation. Integrate `SqlitePool::memory()` with test helpers.

---

### D-010: Documentation drifts from code

Doc comments and markdown files can become stale as APIs evolve. The `route::get()` metadata-only registration is a documented API that silently does nothing — the real handler requires `route::register_handler()`. Developers reading docs find patterns that don't work.

**Fix:** Add documentation tests (`/// ```rust`). Mark deprecated or incomplete APIs with `#[deprecated]`. CI should fail if doc examples don't compile.

---

### D-011: Deployment documentation missing

No documented path from `cargo build` to production. Environment configuration, database migrations, secret management, process supervision (systemd, Docker), and reverse proxy setup are undocumented. Developers must figure out deployment themselves.

**Fix:** Write deployment guide covering Dockerfile, systemd service, env vars, migration strategy, and reverse proxy (nginx/Caddy) configuration.

---

### D-006: No seeder / factory system

Developers have no way to generate test data. They must write SQL by hand or use external tools.

**Fix:** Add `Seeder` trait with `run(conn)`, `make:seeder UserSeeder`, and `artisan db:seed`-like command.

---

### D-007: No HTTP testing helper

Testing endpoints requires spinning up a real server. No `$this->get('/users')` style test client.

**Fix:** Add `TestClient` to viontest that creates mock requests and processes them through the router without a TCP listener.

---

## 🟢 Minor Issues

### m-001: Missing doc comments on public APIs

Many public structs, enums, and methods lack `///` doc comments (`Collection`, `Queue`, `Storage`, `Response` builder methods).

**Fix:** Add doc comments to all public items incrementally.

---

### m-002: READMEs reference 404 GitHub URLs

All README files point to `https://github.com/viontin/docs` which returns 404.

**Fix:** Update or remove URLs.

---

### m-003: No root `.gitignore`

**Fix:** Add root `.gitignore` with standard Rust ignores.

---

### m-004: No `rustfmt.toml` or formatting guarantee

**Fix:** Add `rustfmt.toml`, format entire codebase, enforce in CI.

---

### m-005: Inconsistent naming conventions

Some modules use singular (`controller`, `service`), others plural (`events`, `errors`).

**Fix:** Normalize naming.

---

### m-006: License file exists but no third-party attribution

**Fix:** Run `cargo-deny` to generate license report.

---

### m-007: No error page rendering

Error responses are plain text. No styled error pages for development or production.

**Fix:** Add styled error pages.

---

### m-008: Examples compile with dead-code warnings

`viontin-alpha` compiles with 11 unused-struct warnings.

**Fix:** Use declared items or add `#[allow(dead_code)]` with justification.

---

## 🔵 Suggestions

### S-001: Add hot reload / auto-rebuild for `viontin dev`

Currently `viontin dev` does not watch files. Developer must Ctrl+C and rerun. Integrate with `notify` (already in deps) for file watching.

### S-002: Add debug toolbar for development

Laravel Debugbar-style toolbar showing queries, memory, timing, request data. Injected into HTML responses in development mode.

### S-003: Add project scaffolding (`viontin new --with`)

`viontin new my-app --with auth,api,admin` — generate complete project with auth scaffold, API structure, admin panel.

### S-004: Add tracing/OpenTelemetry

Integrate `tracing` crate for span-based observability across async and sync boundaries.

### S-005: Add metrics with `metrics` crate

Counters, histograms, gauges for requests, queries, cache, queue depth.

### S-006: Add property-based testing with `proptest`

Test core invariants: QueryBuilder SQL generation, cache consistency, event ordering.

### S-007: Add fuzz testing for HTTP parser

The hand-written HTTP/1.1 parser is a critical security surface. Add `cargo-fuzz` targets.

### S-008: Add Redis cache driver

`MemoryCache` and `FileCache` exist but no `RedisCache`. Production deployments need Redis.

### S-009: Add connection pooling to HTTP server

Thread-per-connection is simple but wasteful. Add keep-alive with connection reuse, or document that a reverse proxy (nginx, caddy) is recommended.

### S-010: Add auth scaffolding + proper password hashing

`make:auth` — scaffold login/register/reset-password routes, controllers, and views. Replace `SimpleHasher` with bcrypt or argon2 via a dedicated gem.

### S-011: Add `Gate` / `Policy` authorization system

Middleware-level authorization: `Authorize::new("admin")`, `Policy` trait with `before/after` hooks.

---

## How to Read This Document

Issues are listed in descending severity order. Within each severity level, roughly ordered by impact.

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
