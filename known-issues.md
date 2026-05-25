# Known Issues

> Last updated: 2026-05-25 — C-024, C-027, C-028, C-029 resolved; middleware+mail+ws+server refactored per SoC. Resolved items have been removed for conciseness.

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

**Status:** PARTIALLY FIXED — `.github/workflows/ci.yml` added to framework, viontin, gems, orm, and examples repos. Each runs `cargo check`, `fmt`, `clippy`, `test` on push/PR to `main`.

**Remaining:** CI not yet enabled in GitHub repo settings. No automated test runner configured.

---

### C-021: No background job queue

Only `SyncQueue` exists — jobs execute synchronously in the current thread. No persisted queue, no async processing.

**Status:** Partially fixed — `SyncQueue` now supports delayed jobs via `thread::spawn` with sleep.

**Fix:** Implement `DatabaseQueue` using SQLite, with a background worker thread.

---



### C-022: Only SQLite driver is functional — pg and mysql are stubs

**Location:** `products/orm/crates/pg/`, `products/orm/crates/mysql/`  
**Type:** Missing implementation

PostgreSQL and MySQL drivers exist only as stub files. Applications targeting these databases cannot use Viontin's ORM. Only SQLite works via `rusqlite`.

**Fix:** Implement `Connection` and `ConnectionPool` traits for pg (`tokio-postgres` or `rust-postgres`) and mysql (`sqlx` or `mysql` crate). Minimum: query execution, prepared statements, transaction support.

---

### C-023: No connection pooling for databases

**Location:** `products/orm/crates/orm/`  
**Type:** Missing feature

Every database query opens a new connection. No pool management, no connection reuse, no max-connections limit. This makes any multi-request scenario inefficient.

**Fix:** Implement `ConnectionPool` trait with a real pool (e.g., `deadpool-postgres`, `r2d2`, `sqlx::Pool`). Integrate with the boot lifecycle via a `DatabaseProvider`.

---

### C-024: No password hashing — `SimpleHasher` is XOR-based and insecure

**Status:** RESOLVED — `BcryptHasher` added to `viontin_framework::support` as a default (non-optional) dependency. Implements `Hasher` trait with bcrypt algorithm. `bcrypt` crate is a direct dependency of the framework — no feature flag needed for core security.

---

### C-025: No HTTP test client

**Location:** `products/viontest/`  
**Type:** Missing feature

Testing HTTP endpoints requires starting a real TCP server, binding a port, and making actual HTTP requests. No `TestClient` exists to route requests through the framework in-process.

**Fix:** Add `TestClient` to `viontest` that creates `Request` objects and processes them through the `Router` directly without a TCP listener. Support request assertions (status, headers, body, JSON).

---

### C-026: No Redis cache driver

**Location:** `products/framework/crates/framework/src/cache/`  
**Type:** Missing feature

Only in-process drivers exist (`MemoryCache`, `FileCache`, `NullCache`). No distributed cache via Redis. Applications in production with multiple instances share no cache state.

**Fix:** Add `RedisCache` behind a `cache-redis` feature flag using the `redis` crate. Implement `CacheDriver` trait with connection pooling.

---

### C-027: No SMTP/real mail transport — only `LogTransport`

**Status:** RESOLVED — `SmtpTransport` added to `viontin_framework::mail` behind `smtp` feature flag using `lettre` crate. Supports TLS, authentication, HTML + text bodies, CC. Use: `Mail::smtp("smtp.example.com:587", "user", "pass")`.

---

### C-028: No CORS middleware

**Status:** RESOLVED — `CorsMiddleware` already existed in `viontin_framework::middleware` with `permissive()` and `origin()` constructors, builder methods, preflight handling. Was missing from meta-crate re-exports — fixed by adding to `viontin::CorsMiddleware`. Now use: `boot().middleware(CorsMiddleware::permissive())`.

---

### C-029: No JSON response/request helpers

**Status:** RESOLVED — `Response::json<T: Serialize>(&T) -> Self` and `Request::json<T: DeserializeOwned>(&self) -> Result<T>` added to `viontin_core::http`. `Response::json()` now returns `Response` directly (was `Result<Self, String>`). Serialization errors return a 500 response with an error body.

---

---

## 🟡 Major Issues

### M-010: No structured/JSON logging

**Location:** `products/framework/crates/framework/src/log/`  
**Type:** Missing feature

Logging outputs plain text to stdout only. No structured JSON logging, no log levels configuration at runtime, no log file rotation, no integration with external log aggregators (ELK, Loki, Datadog).

**Fix:** Add `JsonLogChannel` that outputs structured JSON entries. Add configurable log level via `APP_LOG_LEVEL` env var. Support file logging with optional rotation.

---

### M-011: No tracing/span-based observability

**Location:** Entire framework  
**Type:** Missing feature

No `tracing` crate integration. Request spans, database query spans, and async context propagation do not exist. Developers cannot trace a request through the system.

**Fix:** Add `TracingMiddleware` that creates a span per request. Attach `tracing` to the logging system so log entries carry span context. Feature-gate behind `tracing` flag.

---

### M-012: No metrics or `/metrics` endpoint

**Location:** Entire framework  
**Type:** Missing feature

No Prometheus metrics endpoint. No request counters, latency histograms, or cache hit-rate tracking. Operations teams cannot monitor the application.

**Fix:** Add `MetricsMiddleware` that collects request count, duration, status codes. Expose via `/metrics` in Prometheus text format. Feature-gate behind `metrics` flag.

---

### M-013: No health check endpoints (`/healthz`, `/readyz`)

**Location:** `products/framework/`  
**Type:** Missing feature

No built-in health check endpoints. Kubernetes liveness/readiness probes have nothing to query. The roadmap mentions this but no implementation exists.

**Fix:** Add built-in `/healthz` (liveness) and `/readyz` (readiness) endpoints. `readyz` checks database connectivity, cache connectivity, and critical service dependencies. Register automatically when server starts.

---

### M-014: No project Dockerfile or deployment guide

**Location:** Root / docs  
**Type:** Missing infrastructure

No Dockerfile exists in any repo. No deployment documentation. Developers must figure out containerization, database migration strategy, secret management, and process supervision on their own.

**Fix:** Add multi-stage Dockerfile for each product repo. Write `docs/deployment.md` covering Docker, Kubernetes, docker-compose, environment configuration, migration strategy, and reverse proxy setup.

---

### M-015: No model relationships (hasMany, belongsTo, etc.)

**Location:** `products/orm/crates/orm/`  
**Type:** Missing feature

The ORM's QueryBuilder supports SQL queries but provides no relationship handling. Developers must write JOINs manually. No eager loading, no lazy loading, no relationship hydration.

**Fix:** Add `Relationship` trait with `has_one`, `has_many`, `belongs_to`, `belongs_to_many` implementations. Integrate with QueryBuilder for automatic JOIN generation. Add eager loading via `with()` method.

---

### M-016: No database transactions support

**Location:** `products/orm/crates/orm/`  
**Type:** Missing feature

The ORM has no transaction abstraction — no `begin_transaction()`, `commit()`, `rollback()`. Developers must use raw SQL. Any multi-step operation risks partial updates.

**Fix:** Add `begin_transaction()` to `Connection` trait. Add `Transaction` struct with `commit()` and `rollback()`. Integrate with `QueryBuilder` so queries within a transaction scope use the transaction connection.

---

### M-017: No JWT/OAuth/API key authentication

**Location:** `products/framework/crates/framework/src/auth/`  
**Type:** Missing feature

Only `BasicGuard` (HTTP Basic Auth) and `SessionGuard` (session-backed) exist. No JWT token generation/validation, no OAuth2 flows, no API key authentication. Modern API auth is impossible without third-party crates.

**Fix:** Add `JwtGuard` (via `jsonwebtoken` crate), `ApiKeyGuard`, and `OAuthGuard` traits. Provide middleware integration for each. Support token refresh, revocation, and claims validation.

---

### M-018: No S3-compatible file storage driver

**Location:** `products/framework/crates/framework/src/storage/`  
**Type:** Missing feature

Only `LocalStorage` (disk-backed) and `MemoryStorage` exist. No S3-compatible storage (AWS S3, MinIO, DigitalOcean Spaces). Cloud-native deployments requiring shared file storage across instances cannot use the storage facade.

**Fix:** Add `S3Storage` driver (via `aws-sdk-s3` or `rust-s3` crate) implementing the `Driver` trait. Support public/private files, presigned URLs, and directory prefixes.

---

### M-019: No request/file upload handling

**Location:** `products/framework/crates/framework/src/http/`  
**Type:** Missing feature

No multipart form data parsing exists. File uploads are not supported. Developers must manually parse raw bytes from the request body.

**Fix:** Add `multipart` parser (via `multer` crate) to `Request`. Add `UploadedFile` struct with name, size, content type, and temp path. Provide `Request::file()` and `Request::files()` accessors.

---

### M-021: No model factories or database seeders

**Location:** `products/orm/`, `products/viontin/crates/cli/`  
**Type:** Missing feature

Developers have no way to generate test data. No `Seeder` trait, no `Factory` pattern, no `db:seed` command. Writing tests or populating development databases requires manual SQL.

**Fix:** Add `Factory<T>` trait with `definition()` and `make()`/`create()` methods. Add `Seeder` trait with `run(conn)`. Add CLI commands: `make:factory`, `make:seeder`, `db:seed`.

---

### M-022: No query logging/profiling middleware

**Location:** `products/framework/crates/framework/src/db/`  
**Type:** Missing feature

Developers cannot see what SQL queries are executed, how long they take, or how many queries per request. Database optimization is blind.

**Fix:** Add `QueryLogMiddleware` or `DB::listen()` callback that logs queries + duration at debug level. Also expose via `Profiler`-style report accessible per-request.

---

### M-023: No Slack/Discord/Telegram notification channels

**Location:** `products/framework/crates/framework/src/notification/`  
**Type:** Missing feature

Only `MailChannel` and `DatabaseChannel` exist. No webhook-based notification channels for Slack, Discord, or Telegram. Modern team notification workflows are unsupported.

**Fix:** Add `WebhookChannel` trait with Slack, Discord, and Telegram implementations. Each sends structured messages to the respective webhook URL.

---

At least 9 annotations across `project.rs`, `schema.rs`, `check.rs`. Examples: `Blueprint::create`, `Migrator::connection`, `exec_cargo_allow_fail`.

**Fix:** Audit each annotation. Remove unused code or add justified usages.

---

### M-002: `#[allow(dead_code)]` annotations mask unused code

At least 9 annotations across `project.rs`, `schema.rs`, `check.rs`. Examples: `Blueprint::create`, `Migrator::connection`, `exec_cargo_allow_fail`.

**Fix:** Audit each annotation. Remove unused code or add justified usages.

---

### M-003: Global singletons prevent testability

Framework relies on `OnceLock`/`Mutex` singletons (`REGISTRY`, `GLOBAL_LOGGER`, `GLOBAL_CONFIG`, `ROUTE_HANDLERS`, etc.) that cannot be reset between tests.

**Fix:** Migrate to DI via `Application` container. Provide test helpers for isolated instances.

---



### M-006: Multiple Cargo.lock files cause dependency drift

5 separate lockfiles across independent workspaces (viontin, framework, gems, orm, viontest). Since each is a separate git repo with independent dependency trees, this is by design — but cross-repo version alignment (e.g., same `serde` version) is not enforced.

**Fix:** Add cross-repo dependency auditing via `cargo-deny` or a scheduled CI job that checks all repos use compatible versions.

---

### M-007: Edition 2024 constrains MSRV

Requires Rust 1.85+ (stable February 2025). LTS Linux distributions may ship older Rust versions.

**Fix:** Consider `edition = "2021"` or document MSRV prominently.

---

### M-009: `serde` optional in orm but needed for JSON responses

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



---

## 🟢 Minor Issues

### m-001: Missing doc comments on public APIs

Many public structs, enums, and methods lack `///` doc comments (`Collection`, `Queue`, `Storage`, `Response` builder methods).

**Fix:** Add doc comments to all public items incrementally.

---



---

## 🔵 Suggestions



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

### S-012: Add OpenAPI/Swagger documentation generation

Auto-generate OpenAPI 3.0 specs from route definitions and request/response types. Serve via `/openapi.json` or Swagger UI at `/docs`.

### S-013: Add XML / MessagePack / Protobuf serialization

Extend `Response` and `Request` to support `application/xml`, `application/msgpack`, and `application/protobuf` content types via feature-gated implementations.

### S-014: Add API versioning middleware

Middleware that routes requests based on `Accept: application/vnd.api+json;version=1` header or URL prefix (`/v1/`, `/v2/`). Support graceful deprecation with `Sunset` header.

### S-015: Add streaming response support

Support `Transfer-Encoding: chunked` and `text/event-stream` (SSE) for long-lived response streams. Useful for real-time data feeds, log tailing, and large file downloads.

### S-016: Add locale negotiation from HTTP headers

Extend the i18n module to auto-detect locale from `Accept-Language` HTTP header with quality-value parsing and fallback chain. Currently locale must be set programmatically.

### S-017: Add cache tag invalidation and stampede protection

Extend `Cache` facade with tag-based invalidation (`cache.tags(["users", "posts"]).flush()`) and stampede protection (lock-based recomputation for cache-miss hot keys).

### S-018: Add interactive REPL / artisan console

`viontin repl` — interactive shell that loads the application context (DI container, config, database) and allows executing commands, running queries, and inspecting state in real time.

### S-019: Add database migration GUI viewer

`viontin inspect --migrations` — visualize migration status, show pending/ran migrations with timestamps, and provide rollback commands from the CLI.

### S-020: Add real-time notification push (SSE / WebSocket push)

Extend `Notif` facade to support pushing notifications to connected clients via Server-Sent Events or WebSocket. Useful for in-app notifications, toast messages, and live updates.

### S-021: Add image processing support

Feature-gated gem for image resizing, thumbnail generation, format conversion, and EXIF stripping. Integrate with storage facade for upload-then-process workflows.

### S-022: Add email template rendering

Integrate a template engine (e.g., `minijinja`) for rendering HTML emails with layout inheritance, auto-escape, and inline CSS. Provide `Mail::template()` method.

### S-023: Add push notification support (FCM / APNs)

`FcmChannel` and `ApnsChannel` for mobile push notifications. Register devices, send to topics or individual tokens, handle delivery receipts.

### S-024: Add database GUI / admin panel scaffolding

`viontin make:admin` — scaffold a basic admin panel with CRUD for all models, user management, and activity logging. Rendered as HTML with the built-in template engine.

---

## Summary

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| C-005 | Zero test coverage | 🔴 Critical | Open |
| C-011 | All errors are stringly-typed | 🔴 Critical | Open |
| C-013 | No CI/CD pipeline | 🔴 Critical | In Progress |
| C-021 | No background job queue | 🔴 Critical | In Progress |
| C-022 | Only SQLite driver is functional — pg/mysql stubs | 🔴 Critical | Open |
| C-023 | No connection pooling for databases | 🔴 Critical | Open |
| C-024 | No password hashing — `SimpleHasher` is XOR-based | 🔴 Critical | Resolved |
| C-025 | No HTTP test client | 🔴 Critical | Open |
| C-026 | No Redis cache driver | 🔴 Critical | Open |
| C-027 | No SMTP/real mail transport | 🔴 Critical | Resolved |
| C-028 | No CORS middleware | 🔴 Critical | Resolved |
| C-029 | No JSON response/request helpers | 🔴 Critical | Resolved |
| M-002 | `#[allow(dead_code)]` annotations mask unused code | 🟡 Major | Open |
| M-003 | Global singletons prevent testability | 🟡 Major | Open |
| M-006 | Multiple Cargo.lock files cause dependency drift | 🟡 Major | Open |
| M-007 | Edition 2024 constrains MSRV | 🟡 Major | Open |
| M-009 | `serde` optional in orm but needed for JSON responses | 🟡 Major | Open |
| M-010 | No structured/JSON logging | 🟡 Major | Open |
| M-011 | No tracing/span-based observability | 🟡 Major | Open |
| M-012 | No metrics or `/metrics` endpoint | 🟡 Major | Open |
| M-013 | No health check endpoints (`/healthz`, `/readyz`) | 🟡 Major | Open |
| M-014 | No project Dockerfile or deployment guide | 🟡 Major | Open |
| M-015 | No model relationships (hasMany, belongsTo, etc.) | 🟡 Major | Open |
| M-016 | No database transactions support | 🟡 Major | Open |
| M-017 | No JWT/OAuth/API key authentication | 🟡 Major | Open |
| M-018 | No S3-compatible file storage driver | 🟡 Major | Open |
| M-019 | No request/file upload handling | 🟡 Major | Open |
| M-021 | No model factories or database seeders | 🟡 Major | Open |
| M-022 | No query logging/profiling middleware | 🟡 Major | Open |
| M-023 | No Slack/Discord/Telegram notification channels | 🟡 Major | Open |
| D-003 | `make:*` scaffolding does not update `mod.rs` | 🟡 Major | Open |
| D-004 | `make:migration` command missing | 🟡 Major | Open |
| D-006 | No seeder / factory system | 🟡 Major | Open |
| D-008 | Feedback loop — no hot reload | 🟡 Major | Open |
| D-009 | Test setup is too heavy | 🟡 Major | Open |
| D-010 | Documentation drifts from code | 🟡 Major | Open |
| D-011 | Deployment documentation missing | 🟡 Major | Open |
| m-001 | Missing doc comments on public APIs | 🟢 Minor | Open |
| m-005 | Inconsistent naming conventions | 🟢 Minor | Open |
| m-006 | License file exists but no third-party attribution | 🟢 Minor | Open |
| m-007 | No error page rendering | 🟢 Minor | Open |
| S-002 | Add debug toolbar for development | 🔵 Suggestion | Open |
| S-003 | Project scaffolding (`viontin new --with`) | 🔵 Suggestion | Open |
| S-004 | Add tracing/OpenTelemetry | 🔵 Suggestion | Open |
| S-005 | Add metrics with `metrics` crate | 🔵 Suggestion | Open |
| S-006 | Property-based testing with `proptest` | 🔵 Suggestion | Open |
| S-007 | Fuzz testing for HTTP parser | 🔵 Suggestion | Open |
| S-008 | Add Redis cache driver | 🔵 Suggestion | Open |
| S-009 | Add connection pooling to HTTP server | 🔵 Suggestion | Open |
| S-010 | Auth scaffolding + proper password hashing | 🔵 Suggestion | Open |
| S-011 | Add `Gate` / `Policy` authorization system | 🔵 Suggestion | Open |
| S-012 | OpenAPI/Swagger documentation generation | 🔵 Suggestion | Open |
| S-013 | XML / MessagePack / Protobuf serialization | 🔵 Suggestion | Open |
| S-014 | API versioning middleware | 🔵 Suggestion | Open |
| S-015 | Streaming response support | 🔵 Suggestion | Open |
| S-016 | Locale negotiation from HTTP headers | 🔵 Suggestion | Open |
| S-017 | Cache tag invalidation and stampede protection | 🔵 Suggestion | Open |
| S-018 | Interactive REPL / artisan console | 🔵 Suggestion | Open |
| S-019 | Database migration GUI viewer | 🔵 Suggestion | Open |
| S-020 | Real-time notification push (SSE / WebSocket push) | 🔵 Suggestion | Open |
| S-021 | Image processing support | 🔵 Suggestion | Open |
| S-022 | Email template rendering | 🔵 Suggestion | Open |
| S-023 | Push notification support (FCM / APNs) | 🔵 Suggestion | Open |
| S-024 | Database GUI / admin panel scaffolding | 🔵 Suggestion | Open |

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
