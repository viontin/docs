# Service

**Module:** `viontin_framework::service`  
**Trait:** `Service<M: Entity>`  
**Default:** `DefaultService<M, R>`

A **Service** encapsulates business logic between controllers and repositories. It receives a `Repository` via dependency injection and provides `before`/`after` hooks around every operation.

---

## Philosophy

- **Thin controller, fat service** — keep controllers lean by moving business logic here.
- **DI for testability** — inject mock repositories in tests.
- **Hook-based** — `before()` / `after()` wrap every operation for cross-cutting concerns.
- **Compatible with both Model and Entity+Repository** — `Service` works with any `Repository<M>`, whether `M` is a Model or Entity.
- **Gradual adoption** — start with `DefaultService` for zero boilerplate, then implement the trait manually.

---

## Trait Definition

```rust
pub trait Service<M: Entity>: Debug + Send + Sync {
    fn repo(&self) -> &dyn Repository<M>;

    fn before(&self, _action: &str) -> Result<(), String> { Ok(()) }
    fn after(&self, _action: &str) {}

    fn all(&self) -> Result<Vec<M>, String>;
    fn find(&self, id: i64) -> Result<Option<M>, String>;
    fn create(&self, data: Vec<(&str, Value)>) -> Result<i64, String>;
    fn update(&self, id: i64, data: Vec<(&str, Value)>) -> Result<u64, String>;
    fn delete(&self, id: i64) -> Result<u64, String>;
}
```

---

## Usage

### With DefaultService (Zero Boilerplate)

```rust
use viontin_framework::service::DefaultService;

type UserService = DefaultService<User, UserRepo>;

let service = UserService::new(UserRepo { conn: Box::new(conn) });
let users = service.all()?;
```

### Custom Service with Logic

```rust
use viontin_framework::service::Service;
use viontin::entity_system::Entity;
use viontin::repo_system::Repository;
use viontin::fw::db::Value;

pub struct UserService {
    pub repo: UserRepo,
}

impl Service<User> for UserService {
    fn repo(&self) -> &dyn Repository<User> { &self.repo }

    fn before(&self, action: &str) -> Result<(), String> {
        println!("[service] {} begin", action);
        Ok(())
    }

    fn after(&self, action: &str) {
        println!("[service] {} end", action);
    }

    // Custom business logic: only return active users
    fn all(&self) -> Result<Vec<User>, String> {
        self.repo.query()
            .where_eq("active", true)
            .all()
    }

    // Override delete to add soft-delete logic
    fn delete(&self, id: i64) -> Result<u64, String> {
        self.repo.update(id, vec![
            ("deleted_at", Value::Text(chrono::Utc::now().to_string()))
        ])
    }
}
```

### With Hooks for Cross-Cutting Concerns

```rust
impl Service<Order> for OrderService {
    fn before(&self, action: &str) -> Result<(), String> {
        // Authorization check
        if action == "delete" && !current_user_is_admin() {
            return Err("Only admins can delete orders".into());
        }
        Ok(())
    }

    fn all(&self) -> Result<Vec<Order>, String> {
        // Cache check
        if let Ok(cached) = cache_get("orders") {
            return Ok(cached);
        }
        let orders = self.repo().all()?;
        cache_set("orders", &orders);
        Ok(orders)
    }
}
```

---

## DefaultService

`DefaultService` provides a ready-to-use implementation that delegates everything to the repository:

```rust
#[derive(Debug)]
pub struct DefaultService<M: Entity, R: Repository<M>> {
    pub repo: R,
}

impl<M: Entity, R: Repository<M>> DefaultService<M, R> {
    pub fn new(repo: R) -> Self { DefaultService { repo } }
}

impl<M: Entity, R: Repository<M> + 'static> Service<M> for DefaultService<M, R> {
    fn repo(&self) -> &dyn Repository<M> { &self.repo }
    // all(), find(), create(), update(), delete() auto-implemented
}
```

---

## Scaffolding

```bash
viontin make:service UserService
```

Creates `src/services/user_service.rs` with a `Service` implementation template.

---

## See Also

- [Repository](repository) — data access layer
- [Controller](controller) — HTTP handlers using services
- [Entity](entity) — the business objects
