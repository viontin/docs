# ORM
> Last updated: 2026-05-25

**Crate:** `orm`  
**Standalone:** Zero dependencies — does not require `framework`

Multi-driver ORM with a Laravel Eloquent-inspired Query Builder. Pure data access — no patterns, no architecture decisions. Patterns like `Model`, `Entity`, and `Repository` live in `framework`.

---

## Overview

```
┌──────────────────────────────────────────────┐
│  orm (standalone)                     │
│                                               │
│  QueryBuilder  Schema  Migration             │
│                                               │
│  Schema + Blueprint  Migration + Migrator     │
│  Value / Row / DbConfig                       │
│  Connection trait  ConnectionPool trait       │
├──────────────────────────────────────────────┤
│  Driver implementations                      │
│                                               │
│  pg      (PostgreSQL — stub)     │
│  mysql   (MySQL — stub)          │
│  sqlite  (SQLite — implemented)  │
└──────────────────────────────────────────────┘
```

| Crate | Status | Purpose |
|-------|--------|---------|
| `orm` | Implemented | Standalone ORM: QueryBuilder, Schema, Migration, Connection traits, DatabaseType, DriverCapabilities, NoSqlConnection |
| `pg` | **Stub** | PostgreSQL driver (Relational, full SQL) — returns `Err("Not implemented")` |
| `mysql` | **Stub** | MySQL driver (Relational, full SQL) — returns `Err("Not implemented")` |
| `sqlite` | **Implemented** | SQLite driver (Relational, full SQL) — rusqlite-based with bundled SQLite, WAL mode, foreign keys |

---

## Connection & Value Types

### Connection Trait (Multi-Database)

Base trait for **all** database types — SQL, NoSQL, Document, Key-Value, Graph, Search Engine, and more.

```rust
pub trait Connection: Debug + Send + Sync {
    fn driver_name(&self) -> &str;
    fn database_type(&self) -> DatabaseType;
    fn capabilities(&self) -> DriverCapabilities;
    fn query(&self, query: &str, params: &[Value]) -> Result<Vec<Row>, String>;
    fn execute(&self, query: &str, params: &[Value]) -> Result<u64, String>;
    fn last_insert_id(&self) -> Result<i64, String>;
    fn begin(&self) -> Result<(), String>;
    fn commit(&self) -> Result<(), String>;
    fn rollback(&self) -> Result<(), String>;
    fn is_connected(&self) -> bool;
}
```

The `query` parameter is driver-dependent:
- **SQL driver**: SQL string (`"SELECT * FROM users WHERE id = ?"`)
- **MongoDB driver**: JSON filter string (`'{"filter": {"id": 1}}'`)
- **Redis driver**: Command string (`"GET users:1"`)
- **Elasticsearch driver**: JSON query DSL (`'{"query": {"match": {"field": "value"}}}'`)

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
    Null, Int(i64), Float(f64), Text(String), Bool(bool), Blob(Vec<u8>),
}

// From impls:
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

### DatabaseType

Each driver declares its database type:

```rust
pub enum DatabaseType {
    Relational(&'static str),  // pgsql, mysql, sqlite
    Document(&'static str),    // mongodb, couchdb
    KeyValue(&'static str),    // redis, memcached
    Graph(&'static str),       // neo4j
    Search(&'static str),      // elasticsearch
    TimeSeries(&'static str),  // influxdb
    Custom(&'static str),      // other
}
```

### DriverCapabilities

Each driver declares its capabilities:

```rust
pub struct DriverCapabilities {
    pub sql: bool,           // Supports SQL queries
    pub transactions: bool,  // Supports transactions
    pub insert_id: bool,     // Supports last_insert_id
    pub pagination: bool,    // Supports LIMIT/OFFSET
    pub joins: bool,         // Supports JOIN
    pub aggregation: bool,   // Supports GROUP BY / aggregation
    pub batch: bool,         // Supports batch operations
    pub filtering: bool,     // Supports WHERE conditions
    pub ordering: bool,      // Supports ORDER BY
    pub distinct: bool,      // Supports DISTINCT
}
```

Predefined capability profiles:

| Profile | Method | Use Case |
|---------|--------|----------|
| `full_sql()` | `DriverCapabilities::full_sql()` | PostgreSQL, MySQL, SQLite |
| `document_db()` | `DriverCapabilities::document_db()` | MongoDB, CouchDB |
| `key_value()` | `DriverCapabilities::key_value()` | Redis, Memcached |
| `search_engine()` | `DriverCapabilities::search_engine()` | Elasticsearch, Meilisearch |
| `graph_db()` | `DriverCapabilities::graph_db()` | Neo4j, ArangoDB |

### NoSqlConnection

For non-relational databases, implement the `NoSqlConnection` trait alongside `Connection`:

```rust
use viontin_orm::{NoSqlConnection, Value};

impl NoSqlConnection for MyRedisDriver {
    fn get(&self, key: &str) -> Result<Option<Value>, String> {
        let rows = self.query("GET", &[key.into()])?;
        Ok(rows.first().and_then(|r| r.get("value").cloned()))
    }
    fn set(&self, key: &str, value: Value) -> Result<(), String> {
        self.execute("SET", &[key.into(), value])?;
        Ok(())
    }
    fn delete(&self, key: &str) -> Result<bool, String> { .. }
    fn has(&self, key: &str) -> Result<bool, String> { .. }
}
```

### DriverRegistry

Global registry for driver discovery:

```rust
use viontin_orm::{DriverRegistry, DriverInfo, DatabaseType};

let mut registry = DriverRegistry::new();
registry.register(DriverInfo {
    name: "pgsql",
    database_type: DatabaseType::Relational("pgsql"),
    description: "PostgreSQL driver via sqlx",
});

let drivers = registry.by_type(&DatabaseType::Relational("pgsql"));
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

---

> **Patterns (Model, Entity, Repository, Service, Controller)** live in `framework`, not in this crate. See [Model](model), [Entity](entity), [Repository](repository), [Service](service), and [Controller](controller) documentation.

## Query Builder

Laravel Eloquent-style fluent query builder. No model required — works directly with `Connection` and returns `Vec<Row>`.

### Quick Start

```rust
use viontin_orm::{QueryBuilder, Connection};

let conn = get_connection(); // impl Connection

// Fetch all
let users = QueryBuilder::table(&conn, "users").get()?;

// With conditions
let adults = QueryBuilder::table(&conn, "users")
    .where_eq("active", true)
    .where_gt("age", 18)
    .order_by("name", "asc")
    .limit(10)
    .get()?;

// Single row
let user = QueryBuilder::table(&conn, "users")
    .where_eq("email", "alice@example.com")
    .first()?;  // Option<Row>

// Find by ID (uses "id" column by default)
let user = QueryBuilder::table(&conn, "users").find(42)?;

// Count
let count = QueryBuilder::table(&conn, "users")
    .where_null("deleted_at")
    .count()?;  // u64

// Insert
let new_id = QueryBuilder::table(&conn, "users")
    .insert(vec![
        ("name", "Alice".into()),
        ("email", "alice@test.com".into()),
    ])?;  // i64

// Update
let affected = QueryBuilder::table(&conn, "users")
    .where_eq("id", 42)
    .update(vec![("name", "Bob".into())])?;  // u64

// Delete
let deleted = QueryBuilder::table(&conn, "users")
    .where_eq("status", "inactive")
    .delete()?;  // u64

// Pagination
let page = QueryBuilder::table(&conn, "users")
    .order_by("id", "desc")
    .paginate(1, 15)?;  // Page<Row>
```

### Select

```rust
// Specific columns
QueryBuilder::table(&conn, "users")
    .select(&["id", "name", "email"]);

// Add column
QueryBuilder::table(&conn, "users")
    .select(&["id", "name"])
    .add_select("email");

// Distinct
QueryBuilder::table(&conn, "users")
    .distinct()
    .select(&["status"]);
```

### Where Clauses

```rust
// Comparison
.where_eq("age", 25)
.where_not_eq("status", "banned")
.where_gt("age", 18)
.where_gte("score", 50)
.where_lt("age", 65)
.where_lte("score", 100)

// IN / NOT IN
.where_in("role", vec!["admin", "moderator"])
.where_not_in("status", vec!["deleted", "spam"])

// NULL / NOT NULL
.where_null("deleted_at")
.where_not_null("email_verified_at")

// BETWEEN
.where_between("age", 18, 65)
.where_not_between("score", 0, 10)

// Column comparison
.where_column("updated_at", ">", "created_at")

// OR variants
.or_where_eq("role", "superadmin")
.or_where_null("deleted_at")
.or_where_in("status", vec!["pending", "review"])

// Nested groups (WHERE ... AND (... OR ...))
.where_group(|q| {
    q.where_eq("status", "active")
     .or_where_eq("role", "admin")
})

// Raw SQL
.where_raw("YEAR(created_at) = ?", vec![2024])
```

### Joins

```rust
QueryBuilder::table(&conn, "users")
    .join("posts", "users.id", "=", "posts.user_id")
    .left_join("profiles", "users.id", "=", "profiles.user_id")
    .right_join("roles", "users.role_id", "=", "roles.id")
    .cross_join("settings")
    .get()?;
```

### Ordering

```rust
.order_by("name", "asc")
.order_by("created_at", "desc")
.order_by_raw("RAND()")
.latest("created_at")       // ORDER BY created_at DESC
.oldest("created_at")       // ORDER BY created_at ASC
.reorder()                  // clear ordering
```

### Grouping

```rust
.group_by(&["status", "role"])
.having("count", ">", 10)
.or_having("sum", "<", 100)
```

### Limit & Offset

```rust
.limit(10)
.offset(20)
.skip(20)     // alias for offset
.take(10)     // alias for limit
.for_page(2, 15)  // convenience: offset = (page-1)*per_page
```

### Single Value

```rust
// Single column value
let name: Option<Value> = QueryBuilder::table(&conn, "users")
    .where_eq("id", 42)
    .value("name")?;

// Single column list
let names: Vec<Value> = QueryBuilder::table(&conn, "users")
    .where_eq("active", true)
    .pluck("name")?;
```

### Aggregates

```rust
let count: u64 = qb.count()?;
let sum: i64 = qb.sum("votes")?;
let avg: f64 = qb.avg("rating")?;
let min: Option<i64> = qb.min("age")?;
let max: Option<i64> = qb.max("age")?;
let exists: bool = qb.exists()?;
let doesnt: bool = qb.doesnt_exist()?;
```

### Mutations

```rust
// Insert
let id = qb.insert(vec![("name", "Alice".into())])?;

// Update
let affected = qb.where_eq("id", 1).update(vec![("name", "Bob".into())])?;

// Delete
let deleted = qb.where_eq("status", "draft").delete()?;

// Truncate
qb.truncate()?;
```

### Pagination

```rust
// Full pagination (with total count)
let page = QueryBuilder::table(&conn, "users")
    .order_by("id", "desc")
    .paginate(1, 15)?;

page.items;         // Vec<Row>
page.total;         // u64 (total matching records)
page.per_page;      // u64 (15)
page.current_page;  // u64 (1)
page.last_page;     // u64 (total pages)
page.has_more;      // bool

// Simple pagination (no total, for infinite scroll)
let simple = QueryBuilder::table(&conn, "users")
    .simple_paginate(2, 15)?;
```

### Chunking (Memory Efficient)

```rust
QueryBuilder::table(&conn, "users")
    .where_eq("active", true)
    .chunk(100, |rows| {
        for row in &rows {
            // process 100 rows at a time
        }
        Ok(())
    })?;
```

### Conditional Clauses

```rust
QueryBuilder::table(&conn, "users")
    .when(some_condition, |q| {
        q.where_eq("role", "admin")
    })
    .unless(other_condition, |q| {
        q.where_null("deleted_at")
    })
    .get()?;
```

### Debugging

```rust
let qb = QueryBuilder::table(&conn, "users")
    .where_eq("active", true);

qb.to_sql();       // "SELECT * FROM users WHERE active = ?"
qb.to_raw_sql();   // "SELECT * FROM users WHERE active = 1"
qb.dump();         // print SQL to stderr
qb.dd();           // dump and exit
```

---

## Schema & Blueprint

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

### Column Types

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

### Column Modifiers

| Method | Description |
|--------|-------------|
| `.nullable(name)` | Make column nullable |
| `.default(name, value)` | Set default value |
| `.unique(columns)` | Add unique constraint |
| `.index(columns)` | Add index |

### Schema Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `Schema::create(table, fn)` | `(&str, fn) -> Migration` | Create a new table |
| `Schema::table(table, fn)` | `(&str, fn) -> Migration` | Alter existing table |
| `Schema::drop(table)` | `(&str) -> Migration` | Drop a table |
| `Schema::rename(from, to)` | `(&str, &str) -> Migration` | Rename a table |

---

## Migration

```rust
use viontin_orm::Migration;

let migration = Migration::new(
    "create_users_table",
    "CREATE TABLE users (id BIGINT...)",
    "DROP TABLE IF EXISTS users",
);

migration.name;   // "create_users_table"
migration.up;     // SQL to apply
migration.down;   // SQL to revert
```

### Migration Macros

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

### Migrator

```rust
use viontin_orm::Migrator;

let mut migrator = Migrator::new();

migrator.run(vec![
    Schema::create("users", |t| {
        t.id("id");
        t.string("name", 100);
        t.timestamp("created_at");
    }),
])?;

migrator.rollback(vec![migration])?;
```

---

## Driver Crates

Each driver implements `Connection` and `ConnectionPool` from `orm`:

| Crate | Connection | Type | Capabilities | Status |
|-------|-----------|------|--------------|--------|
| `pg` | `PgConnection` | `Relational("pgsql")` | `full_sql()` | **Stub** |
| `mysql` | `MySqlConnection` | `Relational("mysql")` | `full_sql()` | **Stub** |
| `sqlite` | `SqliteConnection` | `Relational("sqlite")` | `full_sql()` | **Stub** |

**NoSQL driver example** (not yet built — showing the pattern):

```rust
use viontin_orm::{Connection, NoSqlConnection, DatabaseType, DriverCapabilities, Value, Row};

struct MongoDriver { /* connection pool */ }

impl Connection for MongoDriver {
    fn driver_name(&self) -> &str { "mongodb" }
    fn database_type(&self) -> DatabaseType { DatabaseType::Document("mongodb") }
    fn capabilities(&self) -> DriverCapabilities { DriverCapabilities::document_db() }
    // query("db.users.find({active: true})", &[]) → Vec<Row>
    // execute("db.users.insert({name: 'Alice'})", &[]) → u64
    fn query(&self, command: &str, params: &[Value]) -> Result<Vec<Row>, String> { todo!() }
    fn execute(&self, command: &str, params: &[Value]) -> Result<u64, String> { todo!() }
    fn last_insert_id(&self) -> Result<i64, String> { todo!() }
    fn begin(&self) -> Result<(), String> { Err("MongoDB doesn't support transactions".into()) }
    fn commit(&self) -> Result<(), String> { Err("Not supported".into()) }
    fn rollback(&self) -> Result<(), String> { Err("Not supported".into()) }
    fn is_connected(&self) -> bool { true }
}

// Optional: implement NoSqlConnection for get/set/delete by key
impl NoSqlConnection for MongoDriver {
    fn get(&self, key: &str) -> Result<Option<Value>, String> {
        // self.query("db.users.find({_id: ObjectId(key)})", &[])
        todo!()
    }
}
```

> All three driver crates are currently stubs. Actual database connectivity (via sqlx for PG/MySQL, rusqlite for SQLite) is planned.

```toml
[dependencies]
# Pick one driver:
pg = { path = "../../products/orm/crates/pg" }
# mysql = { path = "../../products/orm/crates/mysql" }
# sqlite = { path = "../../products/orm/crates/sqlite" }
```

---

## Complete Example

```rust
use viontin_orm::{QueryBuilder, Schema, Migration, Migrator};

// Define migrations
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

// Query without a model — just use the builder directly
fn get_active_users(conn: &dyn viontin_orm::Connection) -> Result<Vec<viontin_orm::Row>, String> {
    QueryBuilder::table(conn, "users")
        .where_eq("active", true)
        .order_by("name", "asc")
        .get()
}

fn find_user(conn: &dyn viontin_orm::Connection, id: i64) -> Result<Option<viontin_orm::Row>, String> {
    QueryBuilder::table(conn, "users").find(id)
}

fn create_user(conn: &dyn viontin_orm::Connection, name: &str, email: &str) -> Result<i64, String> {
    QueryBuilder::table(conn, "users").insert(vec![
        ("name", name.into()),
        ("email", email.into()),
        ("active", true.into()),
    ])
}
```

---

## Standalone Usage (Without Framework)

Since `orm` has zero dependencies, it can be used in any Rust project:

```toml
[dependencies]
orm = { path = "path/to/viontin/products/orm/crates/orm" }
```

```rust
use viontin_orm::{QueryBuilder, Connection, Value, Row};

fn query_example(conn: &dyn Connection) -> Result<(), String> {
    let users = QueryBuilder::table(conn, "users")
        .where_eq("active", true)
        .where_group(|q| {
            q.where_eq("role", "admin")
             .or_where_eq("role", "moderator")
        })
        .order_by("name", "asc")
        .paginate(1, 20)?;

    for user in &users.items {
        let name = user.string("name").unwrap_or_default();
        let email = user.string("email").unwrap_or_default();
        println!("{} <{}>", name, email);
    }

    Ok(())
}
```