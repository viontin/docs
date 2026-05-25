> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Database

**Module:** `viontin_framework::db`

The database module provides the foundation layer for all data access — connection abstractions, value types, and a lightweight SQL query builder. These are built-in types that do not require `orm`.

> **No vendor lock-in:** The standalone `orm` crate is completely optional. Use the framework's built-in db types, or add `orm` separately with `features = ["orm"]` on the `viontin` meta-crate, or use any other ORM entirely.

---

## Architecture

```
┌──────────────────────────────────────┐
│  Application / Service Layer          │
├──────────────────────────────────────┤
│  viontin_framework::db  (built-in)  ←│
│  Connection  ConnectionPool          │
│  QueryBuilder (lightweight)          │
│  Row  Value  DbConfig                │
├──────────────────────────────────────┤
│  Driver Implementations              │
│  (PgConnection / MySqlConnection /   │
│   SqliteConnection / custom)         │
└──────────────────────────────────────┘

Optional: orm (standalone crate)
  ┌──────────────────────────────────┐
  │  QueryBuilder (rich, Eloquent)   │
  │  Schema  Blueprint  Migration    │
  │  DatabaseType  DriverCapabilities│
  │  NoSqlConnection  DriverRegistry │
  └──────────────────────────────────┘
```

---

## Connection

The `Connection` trait defines the interface for database connections:

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

| Method | Description |
|--------|-------------|
| `driver_name()` | Returns driver identifier (`"pgsql"`, `"mysql"`, `"sqlite"`) |
| `query(sql, params)` | Execute SELECT and return rows |
| `execute(sql, params)` | Execute INSERT/UPDATE/DELETE, return affected row count |
| `last_insert_id()` | Get the ID of the last inserted row |
| `begin()` | Start a transaction |
| `commit()` | Commit the current transaction |
| `rollback()` | Roll back the current transaction |
| `is_connected()` | Check if the connection is alive |

---

## ConnectionPool

```rust
pub trait ConnectionPool: Debug + Send + Sync {
    fn config(&self) -> &DbConfig;
    fn connection(&self) -> Result<Box<dyn Connection>, String>;
}
```

| Method | Description |
|--------|-------------|
| `config()` | Return the database configuration |
| `connection()` | Acquire a connection from the pool |

---

## Value

The `Value` enum represents all supported database column types:

```rust
pub enum Value {
    Null,
    Int(i64),
    Float(f64),
    Text(String),
    Bool(bool),
    Blob(Vec<u8>),
}
```

### Automatic Conversions

```rust
Value::from(42)         // i64   → Value::Int(42)
Value::from(3.14)       // f64   → Value::Float(3.14)
Value::from("hello")    // &str  → Value::Text("hello")
Value::from(String::new()) // String → Value::Text(...)
Value::from(true)       // bool  → Value::Bool(true)
```

### Usage in Queries

```rust
// Pass values as query parameters:
let rows = conn.query(
    "SELECT * FROM users WHERE age > ? AND active = ?",
    &[Value::Int(18), Value::Bool(true)],
)?;
```

---

## Row

Represents a single row of results:

```rust
pub struct Row {
    columns: HashMap<String, Value>,
}
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `.get("col")` | `Option<&Value>` | Raw value access |
| `.int("col")` | `Option<i64>` | Parse as integer (also parses from text) |
| `.string("col")` | `Option<String>` | Parse as string |
| `.columns()` | `Vec<String>` | List all column names |

### Examples

```rust
for row in &rows {
    let id: i64 = row.int("id").unwrap();
    let name: String = row.string("name").unwrap_or_default();
    let active: bool = row.get("active").is_some_and(|v| matches!(v, Value::Bool(true)));

    println!("{}: {}", id, name);
}
```

---

## DbConfig

Holds connection configuration:

```rust
pub struct DbConfig {
    pub driver: String,      // "pgsql" | "mysql" | "sqlite"
    pub host: String,        // e.g. "127.0.0.1"
    pub port: u16,           // e.g. 5432
    pub database: String,    // database name
    pub username: String,
    pub password: String,
    pub charset: String,     // default: "utf8"
    pub prefix: String,      // table prefix
    pub pool_min: u32,       // min connections: 2
    pub pool_max: u32,       // max connections: 10
}
```

### Defaults

```rust
DbConfig::default() // or using ..Default::default()
```

| Field | Default |
|-------|---------|
| `driver` | `""` |
| `host` | `"127.0.0.1"` |
| `port` | `0` |
| `database` | `""` |
| `username` | `""` |
| `password` | `""` |
| `charset` | `"utf8"` |
| `prefix` | `""` |
| `pool_min` | `2` |
| `pool_max` | `10` |

---

## QueryBuilder

Fluent SQL query builder that wraps a `Connection`:

```rust
pub struct QueryBuilder<'a> {
    conn: &'a dyn Connection,
    table: String,
    cols: Vec<String>,
    wheres: Vec<WhereClause>,
    order_bys: Vec<(String, String)>,
    limit: Option<u64>,
    offset: Option<u64>,
}
```

### SELECT

```rust
use viontin_orm::QueryBuilder;

let qb = QueryBuilder::table(&conn, "users");

let rows = qb
    .where_eq("age", 25)
    .order_by("name", "ASC")
    .limit(10)
    .offset(0)
    .get()?;    // Vec<Row>
```

Generated SQL:

```sql
SELECT * FROM users WHERE age = ? ORDER BY name ASC LIMIT 10 OFFSET 0
```

### COUNT

```rust
let count = QueryBuilder::table(&conn, "users")
    .where_eq("active", true)
    .count()?;  // u64
```

```sql
SELECT COUNT(*) as count FROM users WHERE active = ?
```

### INSERT

```rust
let id = QueryBuilder::table(&conn, "users")
    .insert(vec![
        ("name", "Alice".into()),
        ("email", "alice@example.com".into()),
        ("active", true.into()),
    ])?;  // last insert id (i64)
```

```sql
INSERT INTO users (name, email, active) VALUES (?, ?, ?)
```

### Method Reference

| Method | Signature | Description |
|--------|-----------|-------------|
| `table(conn, table)` | `(&Connection, &str)` | Create a query builder |
| `where_eq(col, val)` | `(col, val)` | Add `WHERE col = ?` |
| `order_by(col, dir)` | `(col, dir)` | Add `ORDER BY col dir` |
| `limit(n)` | `(u64)` | Add `LIMIT n` |
| `offset(n)` | `(u64)` | Add `OFFSET n` |
| `get()` | `() -> Result<Vec<Row>>` | Execute SELECT |
| `count()` | `() -> Result<u64>` | Execute COUNT |
| `insert(data)` | `(Vec<(&str, Value)>) -> Result<i64>` | Execute INSERT |
| `find(id)` | `(i64) -> Result<Option<Row>>` | Find by primary key |
| `first()` | `() -> Result<Option<Row>>` | First matching row |
| `update(data)` | `(Vec<(&str, Value)>) -> Result<u64>` | Execute UPDATE |
| `delete()` | `() -> Result<u64>` | Execute DELETE |
| `paginate(page, per_page)` | `(u64, u64) -> Result<Page>` | Paginate results |

---

## Transactions

```rust
fn transfer(conn: &dyn Connection, from: i64, to: i64, amount: i64) -> Result<(), String> {
    conn.begin()?;

    let result = (|| {
        conn.execute(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?",
            &[Value::Int(amount), Value::Int(from)],
        )?;

        conn.execute(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            &[Value::Int(amount), Value::Int(to)],
        )?;

        Ok::<_, String>(())
    })();

    match result {
        Ok(()) => conn.commit().map_err(|e| format!("Commit failed: {}", e)),
        Err(e) => {
            conn.rollback().ok();
            Err(format!("Transfer failed: {}", e))
        }
    }
}
```

---

## Driver Implementations

The framework does not ship with built-in drivers. Driver crates implement `Connection` and `ConnectionPool` from `orm`. Each driver must also implement `database_type()` and `capabilities()` for feature discovery:

```rust
fn database_type(&self) -> DatabaseType;      // Relational, Document, KeyValue, etc.
fn capabilities(&self) -> DriverCapabilities;  // sql, transactions, pagination, etc.
```

For NoSQL databases, implement the additional `NoSqlConnection` trait for get/set/delete operations.

| Crate | Connection | ConnectionPool | Status |
|-------|-----------|----------------|--------|
| `pg` | `PgConnection` | `PgPool` | **Stub** — returns `Err("Not implemented")` |
| `mysql` | `MySqlConnection` | `MySqlPool` | **Stub** — returns `Err("Not implemented")` |
| `sqlite` | `SqliteConnection` | `SqlitePool` | **Stub** — returns `Err("Not implemented")` |

> All three driver crates are currently **stubs** with trait implementations that return errors. Actual database connectivity (via sqlx for PG/MySQL, rusqlite for SQLite) is planned.

### Custom Driver

You can implement `Connection` for any type:

```rust
use viontin_orm::{Connection, Value, Row, DbConfig};
```

---

## Complete Example

```rust
use viontin_orm::{Connection, Value, Row, QueryBuilder};

fn list_adults(conn: &dyn Connection) -> Result<Vec<(i64, String)>, String> {
    let rows = QueryBuilder::table(conn, "users")
        .where_eq("active", true)
        .where_gt("age", 18)
        .order_by("name", "ASC")
        .get()?;

    let mut result = Vec::new();
    for row in &rows {
        let id = row.int("id").unwrap_or(0);
        let name = row.string("name").unwrap_or_default();
        result.push((id, name));
    }
    Ok(result)
}

fn create_user(conn: &dyn Connection, name: &str, email: &str) -> Result<i64, String> {
    QueryBuilder::table(conn, "users").insert(vec![
        ("name", name.into()),
        ("email", email.into()),
        ("active", true.into()),
    ])
}

fn count_active(conn: &dyn Connection) -> Result<u64, String> {
    QueryBuilder::table(conn, "users")
        .where_eq("active", true)
        .count()
}
```