> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Database (Query Builder)

**Module:** `viontin_framework::db`

Fluent SQL query builder — construct and execute SQL queries without writing raw SQL strings.

---

## Architecture

```
db/
├── mod.rs       → Value, Row, DbConfig, Connection, ConnectionPool
│                 (re-exported from viontin-core)
├── query_log.rs → Query logging middleware
└── (when `orm` feature is enabled)
    QueryBuilder → Fluent SQL builder
```

---

## Connection

```rust
use viontin::prelude::*;

// A database connection (trait)
pub trait Connection {
    fn query(&self, sql: &str, params: &[Value]) -> Result<Vec<Row>, String>;
    fn execute(&self, sql: &str, params: &[Value]) -> Result<u64, String>;
}

// Values are typed:
let val = Value::Text("hello".into());
let val = Value::Int(42);
let val = Value::Float(3.14);
let val = Value::Bool(true);
let val = Value::Null;
```

---

## QueryBuilder (requires `orm` feature)

```rust
use viontin::prelude::*;

let users = QueryBuilder::new(&conn, "users")
    .select(&["id", "name", "email"])
    .where_eq("active", true)
    .order_by("created_at", "DESC")
    .limit(10)
    .get()?;
```

### Methods

| Method | Description |
|--------|-------------|
| `new(conn, table)` | Start a query on the given table |
| `select(&[col])` | Columns to return (default: `*`) |
| `where_eq(col, val)` | WHERE col = val |
| `where_ne(col, val)` | WHERE col != val |
| `where_gt(col, val)` | WHERE col > val |
| `where_lt(col, val)` | WHERE col < val |
| `where_null(col)` | WHERE col IS NULL |
| `where_not_null(col)` | WHERE col IS NOT NULL |
| `where_raw(sql)` | Raw WHERE clause |
| `order_by(col, dir)` | ORDER BY direction |
| `limit(n)` | LIMIT n rows |
| `offset(n)` | OFFSET n rows |
| `get()` | Execute and return rows |
| `first()` | Execute and return first row |
| `count()` | Execute COUNT query |
| `insert(data)` | INSERT row, return ID |
| `update(data)` | UPDATE matching rows |
| `delete()` | DELETE matching rows |

---

## Raw SQL

```rust
let rows = conn.query("SELECT * FROM users WHERE id = ?", vec![Value::Int(1)])?;
let affected = conn.execute("UPDATE users SET name = ? WHERE id = ?", vec![
    Value::Text("Alice".into()),
    Value::Int(1),
])?;
```

---

## See Also

- [ORM](data/orm) — Full ORM documentation
- [Model](data/model) — Model layer on top of queries
