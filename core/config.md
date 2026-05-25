> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Configuration

**Modules:** `viontin_framework::config`, `viontin_framework::env`

The configuration system has two layers: **environment** (`.env` files, process env vars) and **config** (JSON files with environment-specific overrides).

---

## Environment

### Environment Detection

```rust
use viontin::prelude::*;

let env = Environment::detect();
// Auto-detected from APP_ENV variable
// Defaults to Local if not set
```

| `APP_ENV` | Environment Variant |
|-----------|-------------------|
| `local` / `development` | `Environment::Local` |
| `staging` | `Environment::Staging` |
| `production` | `Environment::Production` |
| `testing` | `Environment::Testing` |
| (anything else) | `Environment::Custom(value)` |
| (not set) | `Environment::Local` |

```rust
env.is_local();         // true
env.is_production();    // false
env.is_testing();       // false
env.as_str();           // "local"
```

### Loading .env Files

```rust
use viontin::prelude::*;

// Load from a specific path
load_env(Path::new(".env"))?;

// Search upward from cwd for .env
load_env_auto()?;
```

Parsing rules:
- Lines are `KEY=VALUE` format
- Comments start with `#`
- Quoted values (`"..."` or `'...'`) have quotes stripped
- Existing environment variables are NOT overwritten

### Environment Helpers

```rust
// Read env vars with defaults
let db_url = env("DATABASE_URL", "postgres://localhost:5432");
let port = env_int("APP_PORT", 3000);
let debug = env_bool("APP_DEBUG", false);

// Check if set
if has_env("S3_KEY") {
    // ...
}
```

---

## Config Files

### Loading

The `ConfigLoader` reads JSON files from a directory with environment-specific overrides:

```rust
use viontin::prelude::*;

let mut loader = ConfigLoader::new("local")  // environment name
    .config_dir("config/");

loader.load()?;

// Initialize global config
config_init(loader.repository().clone());
```

### File Layout

```
config/
├── config.json          # base (always loaded)
├── config.local.json    # overrides for local env
├── config.dev.json      # overrides for dev env
├── config.prod.json     # overrides for production
└── config.testing.json  # overrides for testing
```

Files are loaded in order — overrides merge on top of base:

1. `config.json` → base config
2. `config.{env}.json` → env-specific overrides

### ConfigValue

```rust
#[derive(Debug, Clone)]
pub enum ConfigValue {
    String(String),
    Int(i64),
    Float(f64),
    Bool(bool),
    Array(Vec<ConfigValue>),
    Table(HashMap<String, ConfigValue>),
    Null,
}
```

### Global Access

After initialization, access config globally:

```rust
// With default value
let name: String = config("app.name", "Viontin".into());
let port: i64 = config("app.port", 3000);
let debug: bool = config("app.debug", false);

// Dot notation for nested keys
let host: String = config("database.connections.pg.host", "127.0.0.1".into());
```

```json
{
    "app": {
        "name": "MyApp",
        "port": 8080,
        "debug": true
    },
    "database": {
        "connections": {
            "pg": {
                "host": "127.0.0.1",
                "port": 5432
            }
        }
    }
}
```

### Set at Runtime

```rust
config_set("app.maintenance", ConfigValue::Bool(true));
```

---

## Environment Variable Resolution in Config

Config values can reference environment variables:

```json
{
    "database": {
        "password": ["env:DB_PASSWORD", "default_dev_password"]
    }
}
```

When the config loader encounters an array with exactly 2 elements where the first starts with `env:`, it:
1. Reads the environment variable
2. Falls back to the second element if not set
3. Auto-types the result (string, int, float, bool)

```json
{
    "app": {
        "key": ["env:APP_KEY", ""],
        "mode": ["env:APP_MODE", "development"]
    }
}
```

---

## ConfigRepository (Programmatic)

For cases where you need direct access:

```rust
use viontin_framework::config::{ConfigRepository, ConfigValue};

let mut repo = ConfigRepository::new();

// Set values
repo.set("app.name", ConfigValue::String("MyApp".into()));
repo.set("app.debug", ConfigValue::Bool(true));

// Nested via dot notation
let val = repo.get("app.name");
// Some(&ConfigValue::String("MyApp"))

// Load from JSON string
repo.load_json("database", r#"{"host": "localhost", "port": 5432}"#)?;

// Access nested
let host = repo.get("database.host");
```

---

## ConfigGet Trait

```rust
pub trait ConfigGet: Sized {
    fn from_raw(val: &str) -> Option<Self>;
}
```

Built-in implementations:

| Type | Behavior |
|------|----------|
| `String` | Returns the raw string |
| `i64` | Parses as integer |
| `f64` | Parses as float |
| `bool` | Accepts `true`/`1`/`yes` and `false`/`0`/`no` |

Custom types can implement `ConfigGet`:

```rust
impl ConfigGet for MyEnum {
    fn from_raw(val: &str) -> Option<Self> {
        match val {
            "a" => Some(MyEnum::A),
            "b" => Some(MyEnum::B),
            _ => None,
        }
    }
}
```

---

## Complete Example

```rust
use viontin::prelude::*;
use viontin_framework::config::ConfigLoader;

fn main() {
    // 1. Load .env
    load_env_auto().ok();

    // 2. Detect environment
    let env = Environment::detect();

    // 3. Load config files with env overrides
    let mut loader = ConfigLoader::new(env.as_str())
        .config_dir("config/");
    loader.load().unwrap();
    config_init(loader.repository().clone());

    // 4. Use config
    let app_name: String = config("app.name", "Viontin".into());
    let db_host: String = config("database.host", "localhost".into());
    let db_port: i64 = config("database.port", 5432);
    let debug: bool = config("app.debug", false);

    println!("{} (debug: {})", app_name, debug);
    println!("Database: {}:{}", db_host, db_port);

    // 5. Check environment
    if env.is_local() {
        println!("Running in development mode");
    }
}
```