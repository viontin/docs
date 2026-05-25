# Model

**Module:** `viontin_framework::model`  
**Crate:** `framework` (requires `orm` feature)  
**Trait:** `Model`

**Model** is the recommended default pattern. It combines data (like `Entity`) with persistence (like `Repository`) into a single, convenient trait — similar to Laravel Eloquent's active record.

---

## Philosophy

- **Convenience first** — one trait for data + persistence. The fastest path from zero to running.
- **DI through instance** — `Model::connection(&self)` receives the database connection via the struct instance.
- **Hooks** — `before_save()` / `after_save()` / `before_delete()` / `after_delete()` for customization.
- **No lock-in** — use `Model` by default. Switch to `Entity` + `Repository` when you need clean architecture.

---

## Trait Definition

```rust
pub trait Model: Clone + Debug + Send + Sync + 'static {
    fn connection(&self) -> &dyn Connection;
    fn id(&self) -> String;
    fn table_name() -> &'static str;
    fn primary_key() -> &'static str { "id" }
    fn validate(&self) -> Result<(), Vec<String>> { Ok(()) }
    fn from_row(row: &Row) -> Result<Self, String>;
    fn to_values(&self) -> Vec<(&str, Value)>;

    // Hooks
    fn before_save(&mut self) -> Result<(), String> { Ok(()) }
    fn after_save(&self) -> Result<(), String> { Ok(()) }
    fn before_delete(&self) -> Result<(), String> { Ok(()) }
    fn after_delete(&self) -> Result<(), String> { Ok(()) }

    // Instance CRUD
    fn save(&mut self) -> Result<Self, String>;
    fn delete(&self) -> Result<u64, String>;

    // Static CRUD
    fn all(conn: &dyn Connection) -> Result<Vec<Self>, String>;
    fn find(conn: &dyn Connection, id: i64) -> Result<Option<Self>, String>;
    fn create(conn: &dyn Connection, data: Vec<(&str, Value)>) -> Result<i64, String>;
    fn update(conn: &dyn Connection, id: i64, data: Vec<(&str, Value)>) -> Result<u64, String>;
    fn delete_by_id(conn: &dyn Connection, id: i64) -> Result<u64, String>;

    // Scoped query
    fn query(conn: &dyn Connection) -> QueryBuilder<'_>;
}
```

---

## Usage

### Define a Model

```rust
// Via meta-crate (recommended, requires features = ["orm"])
use viontin::Model;
use viontin::fw::db::{Row, Value, Connection};

#[derive(Debug, Clone)]
pub struct User {
    pub id: i64,
    pub name: String,
    pub email: String,
    pub conn: Box<dyn Connection>,
}

impl Model for User {
    fn connection(&self) -> &dyn Connection { &*self.conn }
    fn id(&self) -> String { self.id.to_string() }
    fn table_name() -> &'static str { "users" }

    fn from_row(row: &Row) -> Result<Self, String> {
        Ok(User {
            id: row.int("id").ok_or("missing id")?,
            name: row.string("name").ok_or("missing name")?,
            email: row.string("email").ok_or("missing email")?,
            conn: todo!("inject your connection"),
        })
    }

    fn to_values(&self) -> Vec<(&str, Value)> {
        vec![
            ("name", self.name.as_str().into()),
            ("email", self.email.as_str().into()),
        ]
    }
}
```

### Fetch Records

```rust
let conn: &dyn Connection = /* get connection */;

// All records
let users = User::all(conn)?;

// Find by ID
let user = User::find(conn, 42)?;

// Scoped query
let admins = User::query(conn)
    .where_eq("role", "admin")
    .order_by("name", "asc")
    .get()?;
```

### Create and Save

```rust
// Create (returns new ID)
let id = User::create(conn, vec![
    ("name", "Alice".into()),
    ("email", "alice@test.com".into()),
])?;

// Instance save (insert or update)
let mut user = User::find(conn, id)?.unwrap();
user.name = "Alice Updated".into();
let saved = user.save()?;  // uses self.connection()
```

### Delete

```rust
// Instance delete
user.delete()?;

// Static delete
User::delete_by_id(conn, 42)?;
```

### Custom Hooks

```rust
impl Model for User {
    fn before_save(&mut self) -> Result<(), String> {
        self.updated_at = chrono::Utc::now().to_string();
        Ok(())
    }

    fn after_save(&self) -> Result<(), String> {
        println!("Saved user {}", self.id);
        Ok(())
    }

    fn validate(&self) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();
        if self.name.is_empty() { errors.push("Name is required".into()); }
        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }
}
```

---

## Model vs Entity + Repository

| Aspect | Model (default) | Entity + Repository |
|--------|----------------|-------------------|
| **Traits to implement** | 1 (`Model`) | 2 (`Entity` + `Repository`) |
| **Persistence** | Built-in (active-record) | Separated (repository pattern) |
| **DI approach** | Connection on self | Connection on repository |
| **Testability** | Mock the connection | Mock the repository |
| **Best for** | Most apps, CRUD, rapid dev | Clean architecture, DDD, large teams |
| **Viontin recommendation** | ✅ Default | ✅ Alternative |

Both are fully compatible. Choose the one that fits your project.

---

## Scaffolding

```bash
viontin make:model User
```

Creates `src/models/user.rs` with a `Model` implementation.

---

## See Also

- [Entity](entity) — Data-only trait (for clean architecture)
- [Repository](repository) — Persistence-only trait (for clean architecture)
- [ORM](orm) — Query Builder and data types
