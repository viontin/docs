# Framework Comparisons

> **Last updated:** 2026-05-24  
> **Note:** Viontin is at version 0.1.0 (experimental). Comparisons reflect current state and intended design goals, not production maturity.

This document compares Viontin with other modern application frameworks. The goal is to help developers understand where Viontin fits in the ecosystem and make informed choices.

---

## Why Compare?

Every framework makes tradeoffs. This comparison is not about "which is better" — it is about understanding design philosophies so you can choose the right tool for your project.

Viontin's design philosophy is inspired by full-stack frameworks like Laravel and Rails, but implemented in Rust with different tradeoffs:

- **Type safety and memory safety** at the language level
- **Zero-cost abstractions** where possible
- **Architectural patterns as opt-in conventions**, not forced structures

---

## Rust Web Frameworks

### Actix-web

**Category:** HTTP framework  
**Version:** 4.x (stable)  
**Repository:** [actix/actix-web](https://github.com/actix/actix-web)

| Dimension | Actix-web | Viontin |
|-----------|-----------|---------|
| **Scope** | HTTP framework only | Full-stack platform (HTTP, CLI, TUI, ORM, scheduler, etc.) |
| **HTTP model** | Async, actor-based, tokio | Sync (thread-per-connection) or optional async |
| **Routing** | Macro-based (`#[get("/")]`) and builder | Builder pattern (`boot().get("/", handler)`) |
| **ORM** | None (bring your own) | Built-in ORM (`viontin-orm`) or BYO |
| **DI** | `web::Data` extractors | TypeId-based container with service providers |
| **Middleware** | Actor-based `Transform` trait | Simple `Middleware` trait with chain |
| **Ecosystem** | Large, many extensions, production-tested | Small, early stage |
| **Performance** | Excellent (actor model, zero-cost) | Moderate (sync threads, simpler architecture) |
| **Learning curve** | Moderate (actor model concepts) | Low (familiar Laravel/Rails patterns) |

**Takeaway:** Actix-web is ideal for high-throughput HTTP APIs where raw performance matters. Viontin is for developers who want a batteries-included platform with CLI tools, ORM, and multiple app types (web, CLI, TUI) from a single dependency.

---

### Axum

**Category:** HTTP framework  
**Version:** 0.8.x (stable)  
**Repository:** [tokio-rs/axum](https://github.com/tokio-rs/axum)

| Dimension | Axum | Viontin |
|-----------|------|---------|
| **Scope** | HTTP framework only | Full-stack platform |
| **HTTP model** | Async, tokio-based, tower ecosystem | Sync (default) or async (optional) |
| **Routing** | Builder with extractors | Builder pattern |
| **Middleware** | Tower `Layer` trait (composable) | Simple `Middleware` trait |
| **DI** | Tower-based service injection | TypeId-based container |
| **Extractors** | Rich type-level extraction (`Path`, `Query`, `Json`, etc.) | Simple `req.param()`, `req.query()` access |
| **Error handling** | Composable via `IntoResponse` | `Result<_, String>` (basic) |
| **Ecosystem** | Large (tokio/tower ecosystem) | Small |
| **Design philosophy** | Modular, composable, async-native | Monolithic, conventional, sync-first |

**Takeaway:** Axum excels in the Rust async ecosystem with its modular, tower-based middleware system and powerful extractors. Viontin prioritizes developer ergonomics with familiar patterns, a unified dependency, and support for non-HTTP app types (CLI, TUI). Axum is a great choice for async API services; Viontin is designed for full-stack applications where a single codebase serves web, CLI, and terminal interfaces.

---

### Rocket

**Category:** HTTP framework  
**Version:** 0.5.x (stable)  
**Repository:** [SergioBenitez/Rocket](https://github.com/SergioBenitez/Rocket)

| Dimension | Rocket | Viontin |
|-----------|--------|---------|
| **Scope** | HTTP framework | Full-stack platform |
| **HTTP model** | Async (tokio) | Sync (default) or async |
| **Routing** | Attribute macros (`#[get("/")]`) | Builder pattern |
| **DI** | Managed state with `&State<T>` | TypeId-based container |
| **ORM support** | None (DIY) | Built-in ORM |
| **Stability** | Mature, well-documented | Experimental, unstable API |
| **Rust features** | Heavily uses proc macros | Minimal proc macros |
| **Learning curve** | Low for HTTP, moderate for Rocket-specific features | Low (familiar conventions) |

**Takeaway:** Rocket pioneered the developer-experience-first approach in Rust web frameworks with its ergonomic macros and state management. Viontin shares the DX-first philosophy but extends it beyond HTTP to encompass the full application lifecycle. Rocket is production-ready for HTTP services; Viontin is broader in scope but earlier in maturity.

---

### Warp

**Category:** HTTP framework  
**Version:** 0.3.x (stable)  
**Repository:** [seanmonstar/warp](https://github.com/seanmonstar/warp)

| Dimension | Warp | Viontin |
|-----------|------|---------|
| **Design** | Composable filters (functional) | Builder + middleware chain |
| **HTTP model** | Async (tokio) | Sync (default) |
| **Routing** | Filter combinators | Path-matching router |
| **Scope** | HTTP only | Full platform |
| **Type safety** | Compile-time filter composition | Runtime middleware chain |
| **Learning curve** | Steep (filter combinators) | Low (imperative builder) |

**Takeaway:** Warp's filter-based composition is elegant for complex routing logic but has a steep learning curve. Viontin's imperative builder is simpler to understand but less composable. Both are valid tradeoffs depending on team experience.

---

### Dioxus

**Category:** UI framework (cross-platform)  
**Version:** 0.6.x (stable)  
**Repository:** [DioxusLabs/dioxus](https://github.com/DioxusLabs/dioxus)

| Dimension | Dioxus | Viontin |
|-----------|--------|---------|
| **Primary domain** | Client-side UI (web, desktop, mobile, TUI) | Server-side application platform (HTTP, CLI, TUI) |
| **Rendering** | Virtual DOM (React-like) | Server-rendered HTML or API JSON |
| **Routing** | Client-side router | Server-side HTTP router |
| **State management** | Hooks + signals | DI container + global state |
| **CLI** | `dx` CLI (build, serve, bundle) | `viontin` CLI (scaffold, dev, inspect) |
| **Backend** | Optional server functions | Primary focus (HTTP, ORM, queues, etc.) |
| **Target platforms** | Web (WASM), Desktop (native), Mobile, TUI | Server (Linux, macOS, Windows), CLI, TUI |
| **Combine with** | Needs a backend framework | Works standalone |
| **Ecosystem** | Growing, active community | Early stage |

**Category difference:** Dioxus and Viontin are not direct competitors — they operate at different layers of the stack:

```
┌─────────────────────────────────────────────────┐
│  Dioxus (UI layer)                               │
│  Client-side rendering, state, routing           │
├─────────────────────────────────────────────────┤
│  REST / GraphQL API                              │
├─────────────────────────────────────────────────┤
│  Viontin (server layer)                          │
│  HTTP, ORM, auth, queues, scheduler, CLI, TUI    │
└─────────────────────────────────────────────────┘
```

Dioxus handles "what users see and interact with" — it renders UI across web, desktop, and mobile using a React-like component model. Viontin handles "what runs on the server" — HTTP routing, database access, authentication, background jobs, CLI tooling, and terminal interfaces.

**Integration:** Dioxus and Viontin can work together in the same Rust codebase. Dioxus serves the frontend (via WASM or server-side rendering), while Viontin provides the backend API, authentication, database access, and CLI tooling. This gives you a full-stack Rust application with a familiar component model on the frontend and a batteries-included backend.

**Takeaway:** If you are building a Rust application with a rich interactive UI, consider Dioxus for the frontend and Viontin for the backend. They complement rather than compete. Dioxus alone needs a backend framework for data persistence, auth, and server logic — Viontin fills that role. Viontin alone generates HTML or JSON on the server — Dioxus provides the interactive client-side layer.

---

### Which Rust Framework Should You Choose?

| If you want... | Choose... |
|---------------|-----------|
| Maximum HTTP throughput | **Actix-web** |
| Modular async ecosystem | **Axum** |
| Polished HTTP DX | **Rocket** |
| Functional composition | **Warp** |
| Full-stack with CLI + ORM + TUI | **Viontin** |

---

## Non-Rust Full-Stack Frameworks

### Laravel (PHP)

**Version:** 12.x (stable)  
**Repository:** [laravel/laravel](https://github.com/laravel/laravel)

| Dimension | Laravel | Viontin |
|-----------|---------|---------|
| **Language** | PHP (interpreted, dynamic) | Rust (compiled, static) |
| **Type safety** | Runtime | Compile-time |
| **Memory model** | Garbage collected | Ownership (no GC) |
| **Boot** | Service providers + facades | Service providers + boot builder |
| **CLI** | Artisan (100+ commands) | Viontin CLI (44 commands) |
| **ORM** | Eloquent (active record) | `viontin-orm` (QueryBuilder + optional Model) |
| **Templating** | Blade (server-rendered) | `html!()` macro (compile-time embed) |
| **Queue** | Redis, database, SQS | SyncQueue (synchronous) |
| **Events** | Pub/sub with listeners | Pub/sub with `EventDispatcher` |
| **Scheduling** | Cron-expression scheduler | Cron-expression `Scheduler` |
| **Notifications** | Multi-channel (mail, SMS, Slack) | `Notif` with MailChannel, DatabaseChannel |
| **Validation** | FormRequest with rules | `FormRequest` trait with rules |
| **Testing** | PHPUnit, Pest | `viontest` (zero-dependency testing) |
| **Package ecosystem** | Packagist (huge) | Gems system (early) |
| **Deployment** | PHP-FPM, Docker | Single binary |
| **Performance** | Moderate (interpreted) | High (compiled native) |
| **Maturity** | Very mature, production-proven | Experimental (0.1.0) |

**Inspiration:** Viontin is heavily inspired by Laravel's architecture — service providers, facades, middleware chains, CLI scaffolding, Eloquent-like QueryBuilder, FormRequest validation, event system, and queue/scheduling patterns all draw from Laravel's well-proven design.

**Key difference:** Laravel's dynamic nature (PHP) allows for powerful metaprogramming but lacks compile-time guarantees. Viontin trades runtime flexibility for Rust's type safety and performance, catching errors at compile time rather than runtime.

**Takeaway:** Laravel is the most mature and productive PHP framework. Viontin aims to bring similar developer experience to Rust, with the benefits of native compilation, type safety, and memory safety. If you are already in the PHP ecosystem, Laravel is the obvious choice. Viontin is for teams that want Rust's performance and safety but miss Laravel's ergonomics.

---

### Ruby on Rails

**Version:** 8.x (stable)  
**Repository:** [rails/rails](https://github.com/rails/rails)

| Dimension | Rails | Viontin |
|-----------|-------|---------|
| **Language** | Ruby (interpreted, dynamic) | Rust (compiled, static) |
| **Convention** | Convention over configuration (strong) | Convention over configuration (opt-in) |
| **CLI** | Rails generators (rich) | Viontin CLI (44 commands) |
| **ORM** | ActiveRecord (active record) | QueryBuilder + optional Model |
| **Routing** | DSL with resources | Builder + route registry |
| **Background jobs** | ActiveJob (multiple backends) | SyncQueue (synchronous) |
| **Testing** | Minitest, RSpec | `viontest` (zero-dependency) |
| **Asset pipeline** | Propshaft, importmap | `html!()` compile-time embedding |
| **Maturity** | Very mature, production-proven | Experimental |
| **Performance** | Moderate (interpreted) | High (compiled native) |

**Takeaway:** Rails defined modern web development conventions. Viontin borrows heavily from Rails' "convention over configuration" philosophy but applies it to the Rust ecosystem. Rails excels at rapid prototyping with minimal code. Viontin offers the same rapid-start experience with Rust's deployment and performance advantages.

---

### Django (Python)

**Version:** 5.x (stable)  
**Repository:** [django/django](https://github.com/django/django)

| Dimension | Django | Viontin |
|-----------|--------|---------|
| **Language** | Python (interpreted, dynamic) | Rust (compiled, static) |
| **Admin** | Built-in admin panel | None (planned via gems) |
| **ORM** | Django ORM (active record) | QueryBuilder + optional Model |
| **DRF** | Django REST Framework | Built-in `Response::json()` |
| **Templating** | Django Templates | `html!()` compile-time embedding |
| **CLI** | `manage.py` commands | Viontin CLI |
| **Maturity** | Very mature, production-proven | Experimental |

**Takeaway:** Django's "batteries included" philosophy is the closest non-Ruby/PHP parallel to Viontin's approach. Both aim to provide everything you need out of the box. Django's admin panel is a standout feature that Viontin does not yet have. Django is ideal for content-heavy applications; Viontin targets systems programming contexts where Rust's strengths matter.

---

### Spring Boot (Java)

**Version:** 3.x (stable)  
**Repository:** [spring-projects/spring-boot](https://github.com/spring-projects/spring-boot)

| Dimension | Spring Boot | Viontin |
|-----------|-------------|---------|
| **Language** | Java (compiled, JVM) | Rust (compiled, native) |
| **DI** | Annotation-based (`@Autowired`) | TypeId-based container |
| **Boot time** | Seconds (JVM warmup) | Milliseconds (native binary) |
| **Memory** | High (JVM heap) | Low (no GC, no runtime) |
| **Configuration** | application.yml + annotations | JSON config + env overlay |
| **CLI** | Spring CLI, Maven/Gradle | Viontin CLI (44 commands) |
| **Microservices** | Spring Cloud ecosystem | ServiceContract + RemoteServiceAdapter |
| **Ecosystem** | Massive (entire Java ecosystem) | Small |
| **Maturity** | Very mature, enterprise-standard | Experimental |

**Takeaway:** Spring Boot is the standard for enterprise Java applications with its mature ecosystem, transaction management, and microservices support. Viontin offers a fraction of the ecosystem but with faster startup, lower memory footprint, and simpler deployment (single binary). Spring Boot is the safe choice for large enterprise teams; Viontin targets smaller teams who value simplicity and performance.

---

### NestJS (TypeScript / Node.js)

**Version:** 11.x (stable)  
**Repository:** [nestjs/nest](https://github.com/nestjs/nest)

| Dimension | NestJS | Viontin |
|-----------|--------|---------|
| **Language** | TypeScript (JS runtime) | Rust (compiled native) |
| **Architecture** | Modular (modules, providers, controllers) | Modular (Module trait, providers, controllers) |
| **DI** | Decorator-based (`@Injectable()`) | TypeId-based container |
| **HTTP** | Express/Fastify under the hood | Hand-written TCP server |
| **WebSocket** | Socket.IO, WS | Hand-rolled WebSocket |
| **CLI** | Nest CLI | Viontin CLI |
| **Testing** | Jest | `viontest` |
| **Performance** | Moderate (JS runtime) | High (compiled native) |
| **Ecosystem** | Large (npm) | Small |

**Takeaway:** NestJS shares Viontin's modular architecture philosophy — modules, controllers, providers, and DI are central to both frameworks. NestJS benefits from the enormous npm ecosystem and TypeScript's type system. Viontin benefits from Rust's true zero-cost abstractions and memory safety.

---

## Comparison Summary

### By Use Case

| Use Case | Best Fit |
|----------|----------|
| High-throughput HTTP API | **Actix-web**, **Axum** |
| Full-stack web application | **Laravel**, **Rails**, **Django** |
| Enterprise microservices | **Spring Boot**, **NestJS** |
| Rust full-stack application | **Viontin** |
| CLI + Web + TUI in one binary | **Viontin** |
| Maximum performance / minimal resources | **Viontin**, **Actix-web** |
| Rapid prototyping | **Rails**, **Laravel**, **Viontin** |

### By Maturity

| Framework | Version | Production Ready | Ecosystem |
|-----------|---------|-----------------|-----------|
| Laravel | 12.x | ✅ Yes | Very large |
| Rails | 8.x | ✅ Yes | Very large |
| Django | 5.x | ✅ Yes | Very large |
| Spring Boot | 3.x | ✅ Yes | Massive |
| NestJS | 11.x | ✅ Yes | Large |
| Actix-web | 4.x | ✅ Yes | Large |
| Axum | 0.8.x | ✅ Yes | Large |
| Rocket | 0.5.x | ✅ Yes | Medium |
| **Viontin** | **0.1.0** | ❌ Experimental | Small |

### By Philosophy

| Philosophy | Frameworks |
|------------|------------|
| **Batteries included** | Laravel, Rails, Django, Spring Boot, **Viontin** |
| **Composable / modular** | Axum, NestJS, Actix-web |
| **Convention over configuration** | Rails, Laravel, **Viontin** |
| **Minimum viable framework** | Express.js, Warp |

---

## Closing Thoughts

Viontin is a young experimental framework. It does not compete with mature, production-proven frameworks on ecosystem, stability, or community size. What it offers is a unique combination:

1. **Rust's safety and performance** — no GC, no runtime, compile-time guarantees
2. **Full-stack scope** — HTTP, CLI, TUI, ORM, scheduler, events, notifications — one dependency
3. **Familiar conventions** — inspired by Laravel, Rails, and NestJS — low learning curve
4. **Zero architecture lock-in** — use Model, Entity+Repository, or QueryBuilder directly
5. **Modular to microservices** — same codebase scales from flat prototype to distributed services

If you are building a Rust application and value developer experience, conventions, and a batteries-included approach, Viontin is worth evaluating. If you need production stability today, established frameworks are the safer choice.

---

## See Also

- [Philosophy](philosophy) — Viontin's design principles
- [Architecture](architecture) — System design
- [Architecture Patterns](architecture-patterns) — RSC, MVC, DDD, Microservices
