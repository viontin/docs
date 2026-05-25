# Entity
> Last updated: 2026-05-25

**Module:** `viontin_framework::entity`  
**Trait:** `Entity`

An **Entity** is a pure data container with identity and validation. It knows nothing about databases.

---

## When to Use

| Pattern | Use Case | Recommended? |
|---------|----------|-------------|
| **Model** (Entity + Repository) | Most applications. Active-record style. | ✅ **Default** |
| **Entity + Repository** | Clean architecture, DDD, testability | ✅ Alternative |
| **QueryBuilder only** | Simple queries, no entity mapping | ✅ Also fine |

**Recommendation:** Use `Model` by default. Switch to `Entity` + `Repository` when you need clean separation of concerns.

Both patterns are fully compatible within the same project — but mixing them for the same entity is not best practice.

---

## Trait Definition

```rust
pub trait Entity: Clone + Debug + Send + Sync + 'static {
    fn id(&self) -> String;
    fn table_name() -> &'static str;
    fn primary_key() -> &'static str { "id" }
    fn validate(&self) -> Result<(), Vec<String>> { Ok(()) }
}
```

| Method | Description | Default |
|--------|-------------|---------|
| `id()` | Unique identifier as string | **Required** |
| `table_name()` | Database table name | **Required** |
| `primary_key()` | Primary key column name | `"id"` |
| `validate()` | Validate before save; return errors to abort | `Ok(())` |

---

## Usage

### Basic Entity

```rust
// Via meta-crate (recommended)
use viontin::Entity;
// Or via framework directly:
// use viontin_framework::entity::Entity;

#[derive(Debug, Clone)]
pub struct User {
    pub id: i64,
    pub name: String,
    pub email: String,
}

impl Entity for User {
    fn id(&self) -> String { self.id.to_string() }
    fn table_name() -> &'static str { "users" }
}
```

### With Validation

```rust
impl Entity for User {
    fn validate(&self) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();
        if self.name.is_empty() { errors.push("Name is required".into()); }
        if !self.email.contains('@') { errors.push("Invalid email".into()); }
        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }
}
```

---

## See Also

- [Model](../data/model) — Recommended default (Entity + persistence combined)
- [Repository](../data/repository) — Data access abstraction (for clean architecture)
- [ORM](../data/orm) — Query Builder and data types