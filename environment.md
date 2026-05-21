# Environment

**Module:** `viontin_framework::env`

Environment detection and `.env` file loading.

---

## Environment Detection

```rust
use viontin::prelude::*;

let env = Environment::detect();
```

Auto-detected from `APP_ENV` variable:

| `APP_ENV` | Variant |
|-----------|---------|
| `local` / `development` | `Environment::Local` |
| `staging` | `Environment::Staging` |
| `production` | `Environment::Production` |
| `testing` | `Environment::Testing` |
| (anything else) | `Environment::Custom(val)` |
| (not set) | `Environment::Local` |

### Methods

```rust
env.is_local();
env.is_production();
env.is_testing();
env.as_str();           // "local", "production", etc.
```

---

## Loading .env

```rust
// Load from a specific path
load_env(Path::new(".env"))?;

// Auto-search upward from cwd
load_env_auto()?;
```

Rules:
- Lines are `KEY=VALUE`
- `#` starts comments
- Quoted values (`"..."` or `'...'`) have quotes stripped
- Existing env vars are NOT overwritten

---

## Environment Helpers

```rust
let db_url = env("DATABASE_URL", "localhost");
let port = env_int("APP_PORT", 3000);
let debug = env_bool("APP_DEBUG", false);

if has_env("S3_KEY") {
    // ...
}
```

| Function | Returns | Default |
|----------|---------|---------|
| `env(key, default)` | `String` | `default` if not set |
| `env_int(key, default)` | `i64` | `default` if not set or unparseable |
| `env_bool(key, default)` | `bool` | `true` for `true`/`1`/`yes`/`on` |
| `has_env(key)` | `bool` | — |

---

## Integration with Config

Environment variables can be referenced in JSON config files:

```json
{
    "database": {
        "password": ["env:DB_PASSWORD", "default_password"]
    }
}
```

See [Configuration](config) for details.

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    // Load .env
    load_env_auto().ok();

    // Detect environment
    let env = Environment::detect();

    // Read env vars
    let host = env("DB_HOST", "localhost");
    let port = env_int("DB_PORT", 5432);
    let debug = env_bool("APP_DEBUG", false);

    if env.is_local() {
        println!("Running in development mode (debug: {})", debug);
    }
}
```
