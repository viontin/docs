# Repository
> Last updated: 2026-05-25

**Module:** `viontin_framework::repository`  
**Trait:** `Repository<M: Entity>`

A **Repository** abstracts data access behind a clean interface. It receives a `Connection` via dependency injection and provides default CRUD operations with lifecycle hooks.

---

## When to Use

| Pattern | Use Case | Recommended? |
|---------|----------|-------------|
| **Model** | Most applications. Active-record style. | ✅ **Default** |
| **Entity + Repository** | Clean architecture, DDD, testability | ✅ Alternative |
| **QueryBuilder only** | Simple queries, no mapping | ✅ Also fine |

**Recommendation:** Use `Model` by default. Reach for `Entity` + `Repository` when you need clean separation of concerns.

Both patterns are fully compatible. Choose the one that fits your project.

---

## Philosophy

- **DI-first** — the repository receives its `Connection` through the `connection()` method, making it testable and swappable.
- **Hook-based** — `before_save` / `after_save` / `before_delete` / `after_delete` hooks allow customization without overriding methods.
- **Optional** — skip the repository entirely and use `QueryBuilder` directly.

---

## Trait Definition

```rust
pub trait Repository<M: Entity>: Debug + Send + Sync {
    fn connection(&self) -> &dyn Connection;
    fn from_row(&self, row: &Row) -> Result<M, String>;
    fn to_values(&self, entity: &M) -> Vec<(&str, Value)>;

    fn table(&self) -> &str { "" }
    fn primary_key(&self) -> &str { "id" }

    // Hooks — all return Ok(()) by default
    fn before_save(&self, _entity: &mut M) -> Result<(), String> { Ok(()) }
    fn after_save(&self, _entity: &M) -> Result<(), String> { Ok(()) }
    fn before_delete(&self, _entity: &M) -> Result<(), String> { Ok(()) }
    fn after_delete(&self, _entity: &M) -> Result<(), String> { Ok(()) }

    // Default CRUD — override any
    fn all(&self) -> Result<Vec<M>, String>;
    fn find(&self, id: i64) -> Result<Option<M>, String>;
    fn create(&self, data: Vec<(&str, Value)>) -> Result<i64, String>;
    fn save(&self, entity: &mut M) -> Result<M, String>;
    fn update(&self, id: i64, data: Vec<(&str, Value)>) -> Result<u64, String>;
    fn delete(&self, entity: &M) -> Result<u64, String>;

    // Scoped query
    fn query(&self) -> QueryScoped<'_, M, Self> where Self: Sized;
}
```

---

## Usage

### Basic Repository

```rust
// Via meta-crate (recommended)
use viontin::Repository;
use viontin::Entity;
use viontin::fw::db::{Row, Value, Connection};

pub struct UserRepo {
    pub conn: Box<dyn Connection>,
}

impl Repository<User> for UserRepo {
    fn connection(&self) -> &dyn Connection { &*self.conn }

    fn from_row(&self, row: &Row) -> Result<User, String> {
        Ok(User {
            id: row.int("id").ok_or("missing id")?,
            name: row.string("name").ok_or("missing name")?,
            email: row.string("email").ok_or("missing email")?,
        })
    }

    fn to_values(&self, entity: &User) -> Vec<(&str, Value)> {
        vec![
            ("name", entity.name.as_str().into()),
            ("email", entity.email.as_str().into()),
        ]
    }
}
```

Now you get `all()`, `find()`, `create()`, `update()`, `delete()` for free:

```rust
let repo = UserRepo { conn: Box::new(conn) };

let all_users = repo.all()?;
let user = repo.find(42)?;
let new_id = repo.create(vec![("name", "Alice".into())])?;
```

### Custom Hook

```rust
impl Repository<User> for UserRepo {
    fn before_save(&self, user: &mut User) -> Result<(), String> {
        user.validate().map_err(|e| e.join(", "))?;
        Ok(())
    }

    fn after_save(&self, user: &User) -> Result<(), String> {
        log::info!("Saved user {}", user.id());
        Ok(())
    }
}
```

### Scoped Query

```rust
let active_admins = repo.query()
    .where_eq("active", true)
    .where_eq("role", "admin")
    .all()?;

let count = repo.query()
    .where_null("deleted_at")
    .count()?;
```

### Override Default

```rust
impl Repository<User> for UserRepo {
    fn all(&self) -> Result<Vec<User>, String> {
        // Custom query — bypass default
        QueryBuilder::table(self.connection(), "users")
            .where_eq("active", true)
            .order_by("name", "asc")
            .get()?
            .into_iter()
            .map(|r| self.from_row(&r))
            .collect()
    }
}
```

---

## Integration with Service

```rust
use viontin_framework::service::{Service, DefaultService};

type UserService = DefaultService<User, UserRepo>;

let service = UserService::new(UserRepo { conn: Box::new(conn) });
let users = service.all()?;
```

---

## Scaffolding

```bash
viontin make:repository UserRepository
```

Creates `src/repositories/user_repository.rs` with `Repository` implementation template.

---

## See Also

- [Entity](entity) — the data objects repositories work with
- [Service](service) — business logic layer on top of repositories
- [ORM](orm) — Query Builder and connection types