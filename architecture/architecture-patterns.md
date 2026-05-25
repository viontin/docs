# Architecture Patterns
> Last updated: 2026-05-25

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

Viontin supports architectural patterns across the entire spectrum — from flat scripts to microservices — **without locking you in**.

| Level | Pattern | When to use |
|-------|---------|-------------|
| 0 | **Flat** | One `main.rs`, prototype, MVP |
| 1 | **MVC** (Model-View-Controller) | Small apps, CRUD, HTML templates |
| 2 | **RSC** (Repository-Service-Controller) | Medium apps, business logic, testability |
| 3 | **Modular Monolith** | Large apps, team boundaries |
| 4 | **Domain-Driven** (DDD) | Complex domains, ubiquitous language |
| 5 | **Microservices** | Independent deployability, scale |

Choose the level that fits your project today. Evolve to the next level when you need it — **without rewriting**.

---

## No Architecture Lock-In

Viontin does not enforce any specific directory structure, trait hierarchy, or pattern.

- Use **flat files** for a prototype — one `main.rs`, no modules.
- Add **controllers** when routes grow.
- Add **services** when business logic needs reuse.
- Add **repositories** when data access needs abstraction.
- Mix patterns within the same project if that makes sense.

Every `make:*` command produces **production-viable code, not stubs**. You decide what to use and when to use it.

---

## Repository-Service-Controller (RSC)

Three-layer separation of concerns:

```
┌──────────────────────────────────────────────┐
│  Controller Layer                             │
│  Handle HTTP requests/responses               │
│  Thin — delegates to services                │
├──────────────────────────────────────────────┤
│  Service Layer                                │
│  Business logic, orchestration, validation    │
│  Depends on repositories                     │
├──────────────────────────────────────────────┤
│  Repository Layer                             │
│  Data access abstraction (database, API, etc) │
│  Wraps QueryBuilder or external clients      │
└──────────────────────────────────────────────┘
```

### When to use

- Medium to large applications
- Multiple data sources (database + API)
- Business logic that needs unit testing without a database
- Teams with clear separation of concerns

### Scaffolding

```bash
viontin make:controller UserController
viontin make:service UserService
viontin make:repository UserRepository
```

### File Layout

```
src/
├── controllers/
│   ├── mod.rs
│   └── user_controller.rs
├── services/
│   ├── mod.rs
│   └── user_service.rs
├── repositories/
│   ├── mod.rs
│   └── user_repository.rs
└── main.rs
```

### Example

```rust
// src/repositories/user_repository.rs
use viontin_orm::{QueryBuilder, Row, Connection};

pub struct UserRepository;

impl UserRepository {
    pub fn all(conn: &dyn Connection) -> Result<Vec<Row>, String> {
        QueryBuilder::table(conn, "users").get()
    }
}

// src/services/user_service.rs
use crate::repositories::user_repository::UserRepository;
use viontin::fw::db::Connection;

pub struct UserService;

impl UserService {
    pub fn list_active_users(conn: &dyn Connection) -> Result<Vec<String>, String> {
        let users = UserRepository::all(conn)?;
        // business logic: filter, transform, validate
        Ok(users.iter().filter_map(|r| r.string("name")).collect())
    }
}

// src/controllers/user_controller.rs
use viontin::{Request, Response, fw::db::Connection};
use crate::services::user_service::UserService;

pub fn index(conn: &dyn Connection, _req: Request) -> Response {
    match UserService::list_active_users(conn) {
        Ok(names) => Response::json(&names).unwrap_or_default(),
        Err(e) => Response::html(&e).status(StatusCode::SERVER_ERROR),
    }
}
```

---

## Model-View-Controller (MVC)

Classic pattern with three roles:

```
┌──────────────────────────────────────────────┐
│  Controller                                   │
│  Handle HTTP, call model, return view         │
├──────────────────────────────────────────────┤
│  Model                                        │
│  Data + business logic                        │
├──────────────────────────────────────────────┤
│  View                                         │
│  HTML templates (compile-time embedded)       │
└──────────────────────────────────────────────┘
```

### When to use

- Smaller applications
- Rapid prototyping
- CRUD-heavy applications
- Teams familiar with Rails/Laravel conventions

### Scaffolding

```bash
viontin make:controller UserController
viontin make:model User
viontin make:view user_index
```

### File Layout

```
src/
├── controllers/
│   ├── mod.rs
│   └── user_controller.rs
├── models/
│   ├── mod.rs
│   └── user.rs
│
views/
└── user_index.html
```

### Example

```rust
// src/models/user.rs
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: i64,
    pub name: String,
    pub email: String,
}

// src/controllers/user_controller.rs
use viontin::{boot, html, Request, Response};
use crate::models::user::User;

pub fn index(_req: Request) -> Response {
    let users = vec![
        User { id: 1, name: "Alice".into(), email: "alice@test.com".into() },
    ];
    // Embed view at compile time, inject data
    let html = html!("views/user_index.html")
        .replace("{{users}}", &serde_json::to_string(&users).unwrap_or_default());
    Response::html(&html)
}
```

---

## Base Classes

Viontin provides optional base classes for each layer with **dependency injection** and **lifecycle hooks**.

| Layer | Trait | DI | Hooks | Scope |
|-------|-------|----|-------|-------|
| **Controller** | `Controller<M>` | `Box<dyn Service<M>>` | `before()`, `after()` | RESTful CRUD actions |
| **Service** | `Service<M>` | `Box<dyn Repository<M>>` | `before()`, `after()` | Business logic orchestration |
| **Repository** | `Repository<M>` | `&dyn Connection` | `before_save()`, `after_save()`, `before_delete()`, `after_delete()` | Data access abstraction |
| **Entity** | `Entity` trait | — | `validate()` | Business data container |
| **FormRequest** | `FormRequest` trait | validation rules | `authorize()`, `validate()` | Request validation |

### 4 Levels of Customization

```
Level 1: Entity + Repository → automatic CRUD with hooks
Level 2: +Service → business logic layer with DI
Level 3: +Controller → RESTful endpoints
Level 4: Skip everything → QueryBuilder directly
```

All base classes are **optional**. Use what you need, skip what you don't.

---

## Comparison

| Aspect | RSC | MVC |
|--------|-----|-----|
| **Separation** | Business logic in services | Business logic in models |
| **Testability** | Easy to mock repositories | Requires database for model tests |
| **Complexity** | More files, more structure | Simpler, fewer layers |
| **Best for** | Large apps, multiple data sources | Small apps, CRUD, prototypes |
| **Data access** | Repository abstraction | Direct QueryBuilder |
| **Views** | Any (API JSON or server-rendered) | HTML templates preferred |

---

## Modular Monolith

Organize your application into self-contained modules, each with its own routes, commands, and providers:

```bash
viontin make:module Billing
```

This scaffolds `src/billing/mod.rs` with a `Module` trait implementation:

```rust
use viontin::module_system::Module;
use viontin::fw::server::Router;

#[derive(Debug, Clone)]
pub struct Billing;

impl Module for Billing {
    fn name(&self) -> &str { "billing" }

    fn routes(&self, router: Router) -> Router {
        router.get("/billing/invoices", Arc::new(list_invoices))
    }
}
```

Register in `main.rs`:

```rust
boot()
    .module(Billing::new())
    .module(Shipping::new())
    .serve(":3000");
```

Modules can be grouped:

```rust
boot()
    .module(ModuleGroup::new("sales")
        .add(Billing::new())
        .add(Shipping::new())
    )
```

> **Extraction path:** A module can be extracted into a microservice by implementing its routes as a separate HTTP server and registering a `RemoteServiceAdapter` in its place.

---

## Domain-Driven Design

Viontin provides first-class DDD building blocks:

| Building Block | Scaffold | Purpose |
|---------------|----------|---------|
| **Domain** | `make:domain Billing` | Bounded context with boundary definition |
| **Aggregate** | `make:aggregate Invoice` | Consistency boundary + event sourcing |
| **Entity** | `make:entity LineItem` | Domain entity with identity |
| **Value Object** | `make:value-object Money` | Immutable value with validation |
| **Domain Event** | `#[domain_event("billing")]` | Typed event for the domain |

### Domain Boundaries

```rust
domain!(billing, allows: [order, payment]);
```

Enforce with:

```bash
viontin check --arch
```

### Event Sourcing

```rust
use viontin::{EventStore, Projection, DomainEvent};

// Store events
event_store.store(&InvoicePaid { invoice_id: "inv_123".into() })?;

// Rebuild projections
projection.rebuild(&event_store)?;
```

### Architecture Testing

```rust
#[test]
fn no_domain_violations() {
    let violations = viontin::check_domains();
    assert!(violations.is_empty(), "Domain violations found");
}
```

---

## Microservices

Viontin enables a clean extraction path from monolith to microservices through **service contracts**.

### Service Contracts

Define an explicit API boundary:

```bash
viontin make:service-contract BillingService
```

This creates `src/contracts/billing_service.rs`:

```rust
use viontin::service_contract::ServiceContract;

#[derive(Debug)]
pub struct BillingService;

impl ServiceContract for BillingService {
    fn name(&self) -> &str { "billing" }
    fn version(&self) -> &str { "1.0" }

    fn handle(&self, command: &str, payload: &[u8]) -> Result<Vec<u8>, String> {
        match command {
            "create-invoice" => { /* ... */ Ok(vec![]) }
            _ => Err(format!("Unknown command: {}", command)),
        }
    }
}
```

### Service Registry

```rust
use viontin::service_contract::{ServiceRegistry, RemoteServiceAdapter};

let mut registry = ServiceRegistry::new();

// In-process (modular monolith)
registry.add(BillingService);

// Resolve and call
let svc = registry.get("billing").unwrap();
let result = svc.handle("create-invoice", b"{}")?;
```

### Extraction Path

```
Single binary (modular monolith)
    │
    ├── Define service contracts
    │
    ├── Extract module as standalone binary
    │     └── Same contract, separate HTTP server
    │
    └── Replace with RemoteServiceAdapter
          └── registry.add(RemoteServiceAdapter::new("billing", "http://billing:8080"))
```

The call site never changes — the same `ServiceContract` trait works for both in-process and remote services.

---

## Choosing and Evolving

You are never locked into either pattern:

```
Prototype (flat)
    │
    ├── Entity + Repository → data layer
    │     └── Add Service → business logic
    │           └── Add Controller → RESTful endpoints
    │
    ├── Add controllers → MVC-lite
    │
    ├── Extract models  → MVC
    │
    ├── Extract services → RSC
    │
    ├── Add repositories → Full RSC
    │
    ├── Organize into Modules → Modular Monolith
    │
    ├── Add Domain boundaries → DDD
    │
    └── Extract ServiceContracts → Microservices
```

Viontin's `make:*` commands scaffold code at any stage. The framework does not enforce, check, or validate which pattern you use. The choice is yours.

> **Rule of thumb:** Start flat, extract when you feel pain. The framework will not get in your way.

---

## See Also

- [Entity](../data/entity) — Business data containers
- [Repository](../data/repository) — Data access abstraction
- [Service](../core/service) — Business logic layer
- [Controller](../core/controller) — HTTP handlers with DI
- [Web App](../app-types/web-app) — HTTP routing, controllers, responses
- [CLI](../dev-tools/cli) — All `make:*` scaffolding commands
- [Getting Started](getting-started) — Build your first app