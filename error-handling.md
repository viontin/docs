> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Error Handling

**Module:** `viontin_framework::error`, `viontin_framework::errors`

Core error types and HTTP error handling.

---

## FrameworkError

```rust
#[derive(Debug, Clone, thiserror::Error)]
pub enum FrameworkError {
    #[error("{0}")]
    Internal(String),
}
```

Unified error type used across all framework modules. Currently has a single `Internal` variant wrapping string messages; planned to be expanded with typed variants (`Config`, `Io`, `Parse`, `Validation`, `Authentication`, `NotFound`, etc.) in future releases.

```rust
use viontin::prelude::*;

fn fallible() -> Result<()> {
    Err(FrameworkError::Internal("something went wrong".into()))
}

fn handle_error() {
    match fallible() {
        Ok(()) => println!("ok"),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

---

## Result

```rust
pub type Result<T> = std::result::Result<T, FrameworkError>;
```

Convenience alias used across the framework:

```rust
use viontin::Result;

fn read_config() -> Result<String> {
    std::fs::read_to_string("config.json")
        .map_err(|e| FrameworkError::Internal(e.to_string()))
}
```

---

## SourceLocation

```rust
pub struct SourceLocation {
    pub file: Option<PathBuf>,
    pub line: usize,
    pub column: usize,
}
```

Tracks where an error originated:

```rust
let loc = SourceLocation {
    file: Some("src/handlers.rs".into()),
    line: 42,
    column: 10,
};

println!("{}", loc);
// "src/handlers.rs:42:10"
```

---

## HTTP Error Handling (Planned)

HTTP-specific error types (`HttpError`, `ErrorReport` with actionable solutions) are planned for future releases. Currently, error responses are constructed manually using `Response::html(...).status(...)`:

```rust
fn handler(_req: Request) -> Response {
    // Database error → 500
    Response::html("<h1>Error</h1><p>Database error</p>").status(StatusCode::SERVER_ERROR)

    // Not found → 404
    Response::html("<h1>Not Found</h1>").status(StatusCode::NOT_FOUND)

    // Validation → 400
    Response::html("<h1>Bad Request</h1>").status(StatusCode::BAD_REQUEST)
}
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn find_user(id: i64) -> Result<String> {
    if id <= 0 {
        return Err(FrameworkError::Internal("invalid id".into()));
    }
    // ... fetch from database
    Ok("Alice".into())
}

fn handler(req: Request) -> Response {
    let id: i64 = req.param("id")
        .and_then(|s| s.parse().ok())
        .unwrap_or(0);

    match find_user(id) {
        Ok(name) => Response::html(format!("<h1>{}</h1>", name)),
        Err(e) => Response::html(format!("<h1>Error</h1><p>{}</p>", e))
            .status(StatusCode::SERVER_ERROR),
    }
}
```
