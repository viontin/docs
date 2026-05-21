> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# ORM

**Crate:** `viontin-orm`  
**Architecture:** Multi-driver with core traits + driver-specific implementations

---

## Overview

The ORM follows a layered architecture:

```
┌──────────────────────────────────────────────┐
│  viontin-orm (core)                           │
│                                               │
│  Model trait    QueryBuilder (typed)          │
│  Schema + Blueprint  Migration + Migrator     │
│  Relations: HasMany, BelongsTo               │
├──────────────────────────────────────────────┤
│  Driver implementations                      │
│                                               │
│  viontin-orm-pg      (PostgreSQL — planned)  │
│  viontin-orm-mysql   (MySQL — planned)       │
│  viontin-orm-sqlite  (SQLite — planned)      │
├──────────────────────────────────────────────┤
│  viontin-framework::db (base layer)           │
│                                               │
│  Connection trait   ConnectionPool trait     │
│  QueryBuilder (raw) Row / Value / DbConfig   │
└──────────────────────────────────────────────┘
```

| Crate | Status | Purpose |
|-------|--------|---------|
| `viontin-orm` | Implemented | Core ORM traits, Schema Blueprint, Migration engine, Relations |
| `viontin-orm-pg` | Placeholder | PostgreSQL driver (via sqlx) |
| `viontin-orm-mysql` | Placeholder | MySQL driver (via sqlx) |
| `viontin-orm-sqlite` | Placeholder | SQLite driver (via rusqlite) |

> **Note:** Driver crates are currently stubs. The core ORM traits and schema builder are fully functional for generating SQL. Actual database connections will be implemented via `sqlx` (PG/MySQL) and `rusqlite` (SQLite).

---

## Framework Base Layer

The `viontin-framework::db` module provides the foundation that all drivers implement:

### Connection Trait

```rust
pub trait Connection: Debug + Send + Sync {
    fn driver_name(&self) -> &str;
    fn query(&self, sql: &str, params: &[Value]) -> Result<Vec<Row>, String>;
    fn execute(&self, sql: &str, params: &[Value]) -> Result<u64, String>;
    fn last_insert_id(&self) -> Result<i64, String>;
    fn begin(&self) -> Result<(), String>;
    fn commit(&self) -> Result<(), String>;
    fn rollback(&self) -> Result<(), String>;
    fn is_connected(&self) -> bool;
}
```

### ConnectionPool Trait

```rust
pub trait ConnectionPool: Debug + Send + Sync {
    fn config(&self) -> &DbConfig;
    fn connection(&self) -> Result<Box<dyn Connection>, String>;
}
```

### Value Types

```rust
pub enum Value {
    Null,
    Int(i64),
    Float(f64),
    Text(String),
    Bool(bool),
    Blob(Vec<u8>),
}

// From impls for ergonomic conversions:
// Value::from(42i64)    → Int(42)
// Value::from("hello")  → Text("hello")
// Value::from(true)     → Bool(true)
```

### Row

```rust
let row: Row = /* from query result */;
row.get("name");       // Option<&Value>
row.int("id");         // Option<i64>
row.string("name");    // Option<String>
row.columns();         // Vec<String>
```

### DbConfig

```rust
DbConfig {
    driver: "pgsql",       // or "mysql", "sqlite"
    host: "127.0.0.1",
    port: 5432,
    database: "myapp",
    username: "root",
    password: "",
    charset: "utf8",
    prefix: "",
    pool_min: 2,
    pool_max: 10,
}
```

### Raw QueryBuilder

The framework provides a raw SQL query builder that the ORM wraps:

```rust
use viontin_framework::db::QueryBuilder;

let qb = QueryBuilder::new(&conn, "users")
    .where_eq("active", true)
    .order_by("name", "ASC")
    .limit(10)
    .offset(0);

let rows = qb.get()?;
let count = qb.count()?;
let id = qb.insert(vec![("name", "Alice".into())])?;
```

---

## ORM Core

### Model

Define a model by implementing the `Model` trait:

```rust
use viontin_orm::Model;
use viontin_framework::db::{Row, Value};

struct User {
    id: i64,
    name: String,
    email: String,
}

impl Model for User {
    fn table_name() -> &'static str { "users" }
    fn primary_key() -> &'static str { "id" }

    fn from_row(row: &Row) -> Result<Self, String> {
        Ok(User {
            id: row.int("id").ok_or("Missing id")?,
            name: row.string("name").ok_or("Missing name")?,
            email: row.string("email").ok_or("Missing email")?,
        })
    }

    fn to_insert_values(&self) -> Vec<(&str, Value)> {
        vec![
            ("name", Value::from(&self.name)),
            ("email", Value::from(&self.email)),
        ]
    }
}
```

#### Built-in Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `table_name` | `() -> &'static str` | Database table name |
| `primary_key` | `() -> &'static str` | Primary key column (default: `"id"`) |
| `from_row` | `(&Row) -> Result<Self, String>` | Hydrate from database row |
| `to_insert_values` | `(&self) -> Vec<(&str, Value)>` | Column-value pairs for INSERT |
| `all` | `(conn) -> Result<Vec<Self>, String>` | Fetch all records |
| `find` | `(conn, id) -> Result<Option<Self>, String>` | Find by primary key |

```rust
// Fetch all users
let users = User::all(&conn)?;

// Find by ID
let user = User::find(&conn, 42)?;
```

---

### QueryBuilder (ORM-Level)

The ORM's `QueryBuilder` wraps the framework's raw query builder with type-safe model hydration:

```rust
use viontin_orm::QueryBuilder;

let qb = QueryBuilder::<User>::new(&conn, "users");

// Chain filters
let active_users = qb
    .where_eq("active", true)
    .order_by("name", "ASC")
    .limit(10)
    .get()?;  // Vec<User>

// First match
let first = qb.where_eq("email", "alice@example.com")
    .first()?;  // Result<Option<User>>

// Count
let count = qb.where_eq("status", "pending")
    .count()?;  // u64

// Insert
let id = qb.create(vec![
    ("name", "Alice".into()),
    ("email", "alice@example.com".into()),
])?;  // last insert id
```

---

### Schema & Blueprint

Define tables using a Laravel-inspired fluent API:

```rust
use viontin_orm::{Schema, Blueprint};

// Create a table
let migration = Schema::create("users", |table| {
    table.id("id");                    // BIGINT AUTO_INCREMENT PRIMARY KEY
    table.string("name", 100);         // VARCHAR(100) NOT NULL
    table.string("email", 255);        // VARCHAR(255) NOT NULL
    table.string("password", 255);
    table.timestamp("created_at");
    table.unique(&["email"]);          // UNIQUE KEY
    table.index(&["name"]);            // INDEX
});
```

#### Column Types

| Method | SQL Type | Description |
|--------|----------|-------------|
| `.id(name)` | `BIGINT UNSIGNED AUTO_INCREMENT` | Primary key column |
| `.increments(name)` | `BIGINT UNSIGNED AUTO_INCREMENT` | Alias for `id()` |
| `.string(name, length)` | `VARCHAR(length)` | Variable string |
| `.text(name)` | `TEXT` | Long text |
| `.integer(name)` | `INTEGER` | Integer |
| `.big_integer(name)` | `BIGINT` | Big integer |
| `.boolean(name)` | `BOOLEAN` | Boolean |
| `.datetime(name)` | `DATETIME` | Date + time |
| `.timestamp(name)` | `TIMESTAMP` | Timestamp |
| `.float(name, precision)` | `FLOAT(precision)` | Float |
| `.decimal(name, precision, scale)` | `DECIMAL(precision, scale)` | Decimal |
| `.json(name)` | `JSON` | JSON data |

#### Column Modifiers

| Method | Description |
|--------|-------------|
| `.nullable(name)` | Make column nullable |
| `.default(name, value)` | Set default value |
| `.unique(columns)` | Add unique constraint |
| `.index(columns)` | Add index |

#### Schema Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `Schema::create(table, fn)` | `(&str, fn) -> Migration` | Create a new table |
| `Schema::table(table, fn)` | `(&str, fn) -> Migration` | Alter existing table |
| `Schema::drop(table)` | `(&str) -> Migration` | Drop a table |
| `Schema::rename(from, to)` | `(&str, &str) -> Migration` | Rename a table |

---

### Migration

A `Migration` is a named change with `up` and `down` SQL:

```rust
use viontin_orm::Migration;

let migration = Migration::new(
    "create_users_table",
    "CREATE TABLE users (id BIGINT... )",
    "DROP TABLE IF EXISTS users",
);

// Or via Schema builder:
let migration = Schema::create("users", |table| {
    table.id("id");
    table.string("name", 100);
    table.timestamp("created_at");
});

migration.name;   // "create_users_table"
migration.up;     // SQL to apply
migration.down;   // SQL to revert
```

#### Migration Macros

```rust
use viontin_orm::{create_table, update_table};

let m1 = create_table!("users", |table| {
    table.id("id");
    table.string("name", 100);
});

let m2 = update_table!("users", |table| {
    table.string("bio", 500);
    table.nullable("bio");
});
```

#### Migrator

The `Migrator` tracks and runs migrations:

```rust
use viontin_orm::Migrator;

let mut migrator = Migrator::new();

// Run pending migrations
migrator.run(vec![
    Schema::create("users", |t| {
        t.id("id");
        t.string("name", 100);
        t.string("email", 255);
        t.timestamp("created_at");
    }),
    Schema::create("posts", |t| {
        t.id("id");
        t.string("title", 200);
        t.text("body");
        t.big_integer("user_id");
        t.timestamp("created_at");
    }),
])?;

// Rollback last migration
migrator.rollback(vec![migration])?;
```

---

### Relations

#### HasMany

```rust
use viontin_orm::HasMany;

// User has many Post
let relation = HasMany::<User, Post>::new("user_id", "id");
```

| Parameter | Description |
|-----------|-------------|
| `foreign_key` | Column on the child table (e.g. `user_id`) |
| `local_key` | Column on the parent table (e.g. `id`) |

#### BelongsTo

```rust
use viontin_orm::BelongsTo;

// Post belongs to User
let relation = BelongsTo::<Post, User>::new("user_id", "id");
```

| Parameter | Description |
|-----------|-------------|
| `foreign_key` | Column on the child table (e.g. `user_id`) |
| `owner_key` | Column on the parent table (e.g. `id`) |

---

## Driver Crate Structure

Each driver crate implements `Connection` and `ConnectionPool` from the framework:

| Crate | Connection | ConnectionPool | Backend |
|-------|-----------|----------------|---------|
| `viontin-orm-pg` | `PgConnection` | `PgPool` | sqlx (planned) |
| `viontin-orm-mysql` | `MySqlConnection` | `MySqlPool` | sqlx (planned) |
| `viontin-orm-sqlite` | `SqliteConnection` | `SqlitePool` | rusqlite (planned) |

```toml
[dependencies]
# Pick one driver:
viontin-orm-pg = { path = "../../repos/orm/crates/viontin-orm-pg" }
# viontin-orm-mysql = { path = "../../repos/orm/crates/viontin-orm-mysql" }
# viontin-orm-sqlite = { path = "../../repos/orm/crates/viontin-orm-sqlite" }
```

```rust
use viontin_orm_pg::PgPool;

let pool = PgPool::new(DbConfig {
    driver: "pgsql".into(),
    host: "127.0.0.1".into(),
    port: 5432,
    database: "myapp".into(),
    ..Default::default()
});
```

---

## Complete Example

```rust
use viontin_orm::{
    Model, QueryBuilder, Schema, Migration, Migrator,
};
use viontin_framework::db::{Row, Value, Connection};

// 1. Define a model
struct User {
    id: i64,
    name: String,
    email: String,
}

impl Model for User {
    fn table_name() -> &'static str { "users" }
    fn primary_key() -> &'static str { "id" }

    fn from_row(row: &Row) -> Result<Self, String> {
        Ok(User {
            id: row.int("id").ok_or("Missing id")?,
            name: row.string("name").ok_or("Missing name")?,
            email: row.string("email").ok_or("Missing email")?,
        })
    }

    fn to_insert_values(&self) -> Vec<(&str, Value)> {
        vec![
            ("name", Value::from(&self.name)),
            ("email", Value::from(&self.email)),
        ]
    }
}

// 2. Define a migration
fn migrations() -> Vec<Migration> {
    vec![
        Schema::create("users", |table| {
            table.id("id");
            table.string("name", 100);
            table.string("email", 255);
            table.unique(&["email"]);
            table.timestamp("created_at");
        }),
    ]
}

// 3. Run migrations
fn run_migrations() -> Result<(), String> {
    let mut migrator = Migrator::new();
    migrator.run(migrations())
}

// 4. Query using the model
fn get_active_users(conn: &dyn Connection) -> Result<Vec<User>, String> {
    QueryBuilder::<User>::new(conn, "users")
        .where_eq("active", true)
        .order_by("name", "ASC")
        .get()
}
```
