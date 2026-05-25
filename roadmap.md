# Roadmap

> **Status:** This document outlines the long-term vision for Viontin across multiple domains. It reflects direction, not commitments. Priorities shift with ecosystem evolution and community feedback.

---

## Current Position: Full-Stack Web Framework

Viontin v0.1.0 is a full-stack web application framework for Rust. This is the **entry point** — the most accessible on-ramp for new users and the foundation upon which everything else is built.

### What Works Today

| Category | Modules |
|----------|---------|
| **HTTP** | Router, sync server, optional async, middleware chain, WebSocket, CSRF |
| **Data** | QueryBuilder (Eloquent-style), SQLite driver, Schema, Migration runner, pagination |
| **Patterns** | Entity, Model, Repository, Service, Controller (all optional, DI-based) |
| **CLI** | 42 scaffold commands, interactive prompts, spinner, progress bar, styling |
| **Security** | Auth guard, session, CORS, rate limiter, CSRF, AES encryption |
| **Messaging** | Events, queue (sync), mail, notifications, scheduler |
| **Infrastructure** | Config (JSON + env), logging, cache, storage, i18n, validation |
| **Dev Tools** | `viontest` (zero-dep testing), `dump`/`dd` debug helpers, profiler, architecture checker |

### Why Start Here

Full-stack web is the most familiar paradigm for developers. Laravel, Rails, Django, and Spring Boot proved that conventions reduce decision fatigue. Viontin follows the same principle but in Rust:

- **Familiar patterns** lower the barrier for developers migrating from other ecosystems
- **Zero lock-in** means projects can evolve from quick prototype to production architecture without rewriting
- **Single dependency** eliminates the "which crate?" paralysis common in Rust

From this foundation, Viontin expands into three directions without fragmenting the codebase.

---

## Three Pillars

```
                    ┌─────────────────────────────────┐
                    │       viontin (meta-crate)      │
                    │    single dependency facade      │
                    └─────────────────────────────────┘
                    │                                 │
     ┌──────────────┴──────────────┐     ┌─────────────┴──────────┐
     │   Pillar 1                  │     │   Pillar 2 & 3         │
     │   Cloud-Native Services     │     │   UI & Game Dev        │
     │   & CLI Toolkit             │     │   (future)             │
     └─────────────────────────────┘     └────────────────────────┘
```

### Pillar 1: Cloud-Native Services & CLI Toolkit

The nearest-term expansion. Viontin's HTTP server, middleware, config, and CLI are already suited for cloud services — they need observability, error hardening, and async maturity.

**Target audience:** Platform engineers, SREs, cloud backend developers, CLI tool authors.

### Pillar 2: BezelUI — Standalone Native UI Framework

The webview gem (`viontin-gem-webview` with wry + tao) provides the foundation. The full UI framework will live in a **standalone project** named [`bezelui`](https://github.com/viontin/bezelui), separate from Viontin core.

BezelUI will start where `viontin-gem-webview` leaves off — adding window management, declarative component DSL, and cross-platform rendering. It will consume Viontin's service layer (config, auth, logging, storage) via the `viontin` meta-crate, but will not depend on Viontin's HTTP server or ORM.

**Target audience:** Desktop application developers, indie developers.

### Pillar 3: Viontin Engine — Standalone Game Engine Foundation

The Entity pattern, event system, and service layer from Pillar 1 can serve as the backend for multiplayer game services. The full game engine will live in a **standalone project** named [`viontin-engine`](https://github.com/viontin/engine), separate from Viontin core.

Viontin Engine will include ECS pattern, `wgpu`-based rendering, audio, physics, and networking — all as optional modules. It will share design patterns with Viontin (Entity, events, DI) but will not depend on Viontin's HTTP infrastructure.

**Target audience:** Indie game developers, Rust game dev community.

---

## Direction by Domain

### 1. Observability (Nearest)

| Goal | Approach |
|------|----------|
| Request tracing | `tracing` crate integration. `TracingMiddleware` creates a span per request. Auto-propagates span context across async boundaries. |
| Metrics | `metrics` crate integration. Auto-collect request count, latency, status codes. Expose via `/metrics` endpoint in Prometheus format. |
| Structured logging | `tracing-subscriber` for JSON log output. Replace `println!` in server code with structured events. |
| OpenTelemetry | Optional export via `opentelemetry-otlp` for tracing and metrics to collectors like Jaeger or Grafana. |

The `RequestId` middleware already exists. The next step is attaching a `tracing` span to each request and routing log events through the span.

### 2. Typed Errors

Every layer currently returns `Result<_, String>`. This must become proper error types:

| Error Type | Variants |
|------------|----------|
| `RepositoryError` | `NotFound`, `ConnectionLost`, `QueryFailed { sql, detail }`, `ConstraintViolation` |
| `ServiceError` | `Validation(Vec<String>)`, `NotFound`, `Forbidden`, `Repository(RepositoryError)` |
| `HttpError` | Maps from ServiceError to HTTP status codes (404, 422, 500, etc.) |
| `FrameworkError` | Replace `Internal(String)` with typed variants for all framework subsystems |

Each error type implements `std::error::Error`, `Display`, and has `From` conversions between layers.

### 3. Async Growth

| Goal | Approach |
|------|----------|
| First-class async | Make the async server (Tokio-based) the default. Sync server becomes an optional feature for minimal environments. |
| Connection pooling | HTTP keep-alive with connection reuse. Database connection pool management integrated with the ORM. |
| Graceful shutdown (done) | SIGTERM/SIGINT handler + in-flight request drain. Already implemented behind the `shutdown` feature. |

### 4. Service Mesh Patterns

Patterns common in cloud-native microservices, provided as optional middleware:

| Pattern | Description |
|---------|-------------|
| **Retry** | Middleware that retries failed requests with exponential backoff |
| **Circuit Breaker** | Prevents cascading failures by stopping requests to unhealthy services |
| **Timeout (done)** | Per-request timeout middleware. Basic version already implemented. |
| **Rate Limit (done)** | Token bucket rate limiter + middleware. Already implemented. |
| **Service Discovery** | Resolve downstream service URLs from a registry (static, DNS, or Consul) |

### 5. CLI Toolkit Maturity

| Feature | Description |
|---------|-------------|
| Shell completion | Generate bash, zsh, and fish completion scripts from command signatures |
| Paginated output | Table and list output with auto-pagination for large result sets |
| Progress & spinner (done) | Already exists. Polish animation, add determinate progress for file operations |
| Prompt history | Remember previous inputs across invocations |
| Plugin execution | Load and run WASM-compiled gems as CLI subcommands |
| `viontin dev` hot reload | File watching via `notify` crate (already in deps) — auto-restart on source changes |

### 6. Native UI Framework (Future)

| Stage | Capability |
|-------|------------|
| **L0 — Webview (done)** | `viontin-gem-webview` with wry + tao. `launch_with_ipc()` for Rust ↔ JS communication. |
| **L1 — Window management** | Multiple windows, menu bar, system tray, dialog boxes, file picker, drag-and-drop |
| **L2 — Declarative UI DSL** | Macro-based component tree (React-like or Flutter-like) that renders to native widgets via wry or direct GPU |
| **L3 — Cross-platform framework** | Compete in the same space as Tauri. Desktop + mobile + web from a single codebase. |

The service layer from Pillar 1 (auth, storage, config, logging) serves as the backend for any desktop app built with Pillar 2.

### 7. Game Engine Foundation (Future)

| Stage | Capability |
|-------|------------|
| **L0 — Game services** | Auth, matchmaking, leaderboards, asset pipelines, and player state — all built on Pillar 1's service layer |
| **L1 — ECS pattern** | Entity-Component-System integrated with `viontin::app` boot lifecycle. `Entity` trait and `EventStore` from the DDD module provide the foundation. |
| **L2 — Rendering gem** | Optional `wgpu`-based rendering gem for 2D and 3D graphics |
| **L3 — Full engine** | Scene graph, physics (rapier), audio (cpal), networking (QUIC), and an optional editor |

The Entity pattern and event sourcing from the current domain module already align with ECS thinking.

---

## Project Map

```
┌─────────────────────────────────────────────────────────────┐
│  viontin (meta-crate)                                        │
│  Full-stack web + cloud-native services + CLI toolkit       │
│  github.com/viontin/framework                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  viontin-gem-webview  ──→  bezelui (standalone)              │
│  Webview + IPC bridge        Native UI framework             │
│  (in viontin/gems repo)      github.com/viontin/bezelui     │
│                                                              │
│  viontin (patterns)  ──→  viontin-engine (standalone)        │
│  Entity, events, DI           ECS + rendering + physics      │
│  (in viontin/framework)       github.com/viontin/engine     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

All three projects share design philosophy and patterns. Bezelui and Viontin Engine consume Viontin as a dependency but are independent codebases.

---

## Dependency Flow Between Pillars

```
Phase 0 — Full-Stack Web (current)
    │
    ├── provides: HTTP, ORM, CLI, middleware, patterns
    ├── provides: DI container, config, logging, events
    │
    ├──→ Pillar 1: Cloud-Native Services
    │     ├── needs: typed errors, observability, async
    │     └── reuses: HTTP, middleware, DI, config, logging
    │
    ├──→ Pillar 2: Native UI
    │     ├── needs: window management, GPU, event loop
    │     └── reuses: DI, config, logging, storage, events
    │
    └──→ Pillar 3: Game Engine
          ├── needs: ECS, rendering, physics, audio
          └── reuses: Entity pattern, events, DI, config, logging
```

Each pillar shares the foundation (meta-crate, DI, config, logging, events, gems system) without requiring the others.

---

## Design Principles (All Phases)

| Principle | Rationale |
|-----------|-----------|
| **Single meta-crate** | One dependency unlocks the entire platform. Users opt into specific features via Cargo feature flags. |
| **Optional patterns** | Entity, Model, Repository, Service, Controller — all optional. Use what you need, skip what you don't. |
| **Gems for small extensions** | Small integrations (Tailwind, Inertia) live in gems within the monorepo. Large domains (UI framework, game engine) become standalone projects that consume Viontin as a dependency. |
| **Zero proc macros (almost)** | Only `#[domain]` and `#[domain_event]` exist, both behind the `domain` feature flag. No derive macros. |
| **Synchronous by default** | Sync code is simpler to write, debug, and teach. Async is available behind a feature flag for when it's needed. |
| **No architecture lock-in** | Flat → MVC → RSC → Modular → DDD → Microservices — same codebase, no rewrites. |

---

## Success Criteria

Each directional goal is considered "ready" when:

| Domain | Success Looks Like |
|--------|-------------------|
| **Cloud-Native** | A developer can deploy a Viontin service to Kubernetes with Prometheus metrics, structured logging, and health checks — without adding any external crate. |
| **CLI Toolkit** | A developer can build and distribute a CLI tool with interactive prompts, shell completion, and progress reporting — in under 50 lines of code. |
| **Native UI** | A developer can create a desktop window with native menus, system tray, and a webview frontend — using the same DI, config, and auth as their HTTP services. |
| **Game Engine** | A developer can build a multiplayer game server using Viontin's ECS, networking, and rendering gems — with the same logging, metrics, and deployment patterns as their web services. |

---

## What This Roadmap Is Not

- **Not a release schedule.** There are no version numbers, deadlines, or milestones attached to any direction here.
- **Not a promise.** Priorities change. Ecosystem shifts happen. Some directions may never materialize.
- **Not a commitment to backward compatibility.** Viontin is v0.1.0 and experimental. Breaking changes are expected.

---

## See Also

- [Architecture](architecture) — Current system design
- [Known Issues](known-issues) — What needs to be fixed before any direction can advance
- [Philosophy](philosophy) — Design principles that guide all directions
- [Comparisons](comparisons) — How Viontin relates to other frameworks and tools
