> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Path Utilities

**Module:** `viontin_framework::path`

Project root detection, path resolution, and URL formatting.

---

## base_path

Resolve a path relative to the project root (directory containing `Cargo.toml`):

```rust
use viontin::prelude::*;

let config = base_path("config/config.json");
// "/home/user/project/config/config.json"

let root = base_path("");
// "/home/user/project"
```

Searches upward from the current working directory for `Cargo.toml`.

---

## base_path_glob

Glob pattern matching relative to the project root:

```rust
use viontin::prelude::*;

let files = base_path_glob("src/**/*.rs").unwrap();
for file in &files {
    println!("{}", file.display());
}
```

Returns a sorted `Vec<PathBuf>`.

---

## url

Format a path as a URL with leading slash:

```rust
use viontin::prelude::*;

url("users");         // "/users"
url("/users");        // "/users"
url("/users/:id");    // "/users/:id"
url("");              // "/"
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn load_config() -> Result<String, String> {
    let path = base_path("config/app.json");
    fs::read(&path)
}

fn find_templates() -> Result<Vec<std::path::PathBuf>, String> {
    base_path_glob("templates/**/*.html")
}

fn register_routes() {
    // Use url() for consistent route paths
    route::get(url("api/users"), "list_users", "src/routes.rs:5");
    route::post(url("api/users"), "create_user", "src/routes.rs:6");
}
```
