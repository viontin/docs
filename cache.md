> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Caching

**Module:** `viontin_framework::cache`  
**Facade:** `Cache`

Caching system with swappable drivers, TTL support, and a facade pattern.

---

## Architecture

```
Cache (facade)
  │
  ├── key prefixing
  ├── remember (get-or-set)
  ├── pull (get-and-delete)
  │
  └── Box<dyn CacheDriver>
        │
        ├── MemoryCache   — in-memory HashMap (fastest, ephemeral)
        ├── FileCache     — file-backed (persistent across restarts)
        └── NullCache     — no-op (for testing)
```

---

## Quick Start

```rust
use viontin::prelude::*;

// In-memory cache (development)
let cache = Cache::memory();

// File cache (production)
let cache = Cache::file("storage/cache/");

// Null cache (testing)
let cache = Cache::null();

// Use
cache.set("key", "value", Some(3600));  // TTL: 1 hour
let val = cache.get("key");             // Some("value")
cache.has("key");                       // true
cache.delete("key");                    // remove (alias: forget)
cache.clear();                          // remove all
```

---

## Cache Facade

The `Cache` struct wraps a `Box<dyn CacheDriver>` with additional features.

### Creating a Cache

```rust
// Shorthand constructors:
Cache::memory()             // In-memory
Cache::file("path/to/dir")  // File-backed
Cache::null()               // No-op

// With custom driver:
use viontin_framework::cache::{Cache, CacheDriver, MemoryCache};

Cache::with_driver(MemoryCache::new())
```

### Key Prefixing

```rust
let cache = Cache::memory()
    .with_prefix("app");   // all keys prefixed with "app:"

cache.set("users_count", "1500", None);
// internally stored as "app:users_count"
```

### Methods

```rust
fn get(&self, key: &str) -> Option<String>
fn set(&self, key: &str, value: &str, ttl: Option<u64>)
fn delete(&self, key: &str)
fn has(&self, key: &str) -> bool
fn clear(&self)
fn remember(&self, key: &str, ttl: u64, f: impl FnOnce() -> String) -> String
fn pull(&self, key: &str) -> Option<String>
fn increment(&self, key: &str, amount: i64) -> i64
```

| Method | Description |
|--------|-------------|
| `get(key)` | Retrieve a value |
| `set(key, value, ttl)` | Store a value with optional TTL |
| `delete(key)` | Remove a key (also aliased as `forget` in Laravel-style usage) |
| `has(key)` | Check if key exists and is not expired |
| `clear()` | Remove all keys |
| `remember(key, ttl, fn)` | Get or set via closure |
| `pull(key)` | Get and delete atomically |
| `increment(key, amount)` | Atomic increment |

---

## CacheDriver Trait

Implement for custom cache backends:

```rust
pub trait CacheDriver: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn get(&self, key: &str) -> Option<String>;
    fn set(&self, key: &str, value: &str, ttl_seconds: Option<u64>);
    fn delete(&self, key: &str);
    fn clear(&self);
    fn has(&self, key: &str) -> bool { self.get(key).is_some() }
    fn increment(&self, key: &str, amount: i64) -> i64;
    fn decrement(&self, key: &str, amount: i64) -> i64 { self.increment(key, -amount) }
}
```

---

## Built-in Drivers

### MemoryCache

In-memory storage using `HashMap` behind a `Mutex`. Fastest option — data is lost on process restart.

```rust
Cache::memory()
// or
Cache::with_driver(MemoryCache::new())
```

Best for:
- Development
- Single-process applications
- Temporary / non-critical data

### FileCache

Persists cache entries as individual files on disk. Each entry is stored as `<sanitized_key>.cache`.

```rust
Cache::file("storage/cache/")
// or
Cache::with_driver(FileCache::new("storage/cache/"))
```

File format:
```
<expiry_timestamp>
<value>
```

Best for:
- Production with file system available
- Data that should survive restarts
- Single-server deployments

### NullCache

No-op implementation — every operation is a no-op. Always returns `None` for reads.

```rust
Cache::null()
// or
Cache::with_driver(NullCache)
```

Best for:
- Testing
- Disabling cache without code changes
- Feature flags

---

## remember Pattern

The `remember` method implements get-or-set:

```rust
let users = cache.remember("users_list", 300, || {
    // This closure runs only on cache miss
    let result = fetch_users_from_database();
    result  // must return String
});
```

Equivalent to:

```rust
let users = cache.get("users_list").unwrap_or_else(|| {
    let result = fetch_users_from_database();
    cache.set("users_list", &result, Some(300));
    result
});
```

---

## pull Pattern

Retrieve and delete in one operation:

```rust
if let Some(token) = cache.pull("reset_token") {
    // Token consumed — cannot be used again
    verify_reset_token(&token);
}
```

---

## increment / decrement

```rust
cache.set("counter", "10", None);
cache.increment("counter", 1);    // 11
cache.increment("counter", 5);    // 16
cache.decrement("counter", 3);    // 13
```

---

## TTL (Time To Live)

TTL values are in seconds:

```rust
cache.set("key", "value", Some(3600));      // expires in 1 hour
cache.set("key", "value", Some(86400));     // expires in 1 day
cache.set("key", "value", None);            // never expires
```

- `MemoryCache` checks expiry on read
- `FileCache` checks expiry on read and cleans expired files
- `NullCache` ignores TTL

---

## Custom Driver

```rust
use viontin_framework::cache::{Cache, CacheDriver};

#[derive(Debug)]
struct RedisCache {
    // your connection
}

impl CacheDriver for RedisCache {
    fn name(&self) -> &str { "redis" }
    fn get(&self, key: &str) -> Option<String> { /* ... */ }
    fn set(&self, key: &str, value: &str, ttl: Option<u64>) { /* ... */ }
    fn delete(&self, key: &str) { /* ... */ }
    fn clear(&self) { /* ... */ }
    fn increment(&self, key: &str, amount: i64) -> i64 { /* ... */ }
}

let cache = Cache::with_driver(RedisCache::new());
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    let cache = Cache::file("storage/cache/")
        .with_prefix("myapp");

    // Remember with TTL
    let version = cache.remember("app_version", 60, || {
        std::env::var("APP_VERSION").unwrap_or("unknown".into())
    });
    println!("Version: {}", version);

    // Atomic counter
    let visits = cache.increment("page_visits", 1);
    println!("Visit count: {}", visits);

    // Conditional
    if !cache.has("setup_complete") {
        run_setup();
        cache.set("setup_complete", "true", None);
    }

    if let Some(data) = cache.pull("one_time_token") {
        process_with_token(&data);
    }
}
```
