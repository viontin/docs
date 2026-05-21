# Rate Limiting

**Module:** `viontin_framework::rate`  
**Facade:** `RateLimiter`  
**Driver Trait:** `RateLimiterDriver`  
**Implementation:** `TokenBucketLimiter`

Laravel-style rate limiter facade with token bucket backend.

---

## Quick Start

```rust
use viontin::RateLimiter;

// Attempt with callback — only runs if under limit
let result = RateLimiter::attempt("login:42", 5, 60, || {
    perform_login()
});

match result {
    Ok(response) => response,
    Err(()) => Response::html("Too many attempts").status(429),
}

// Or check manually
if RateLimiter::too_many_attempts("login:42", 5) {
    let secs = RateLimiter::available_in("login:42");
    return Response::html(format!("Retry in {} seconds", secs)).status(429);
}

RateLimiter::hit("login:42", 60); // increment counter
```

---

## RateLimiter Facade

### Instance Methods

```rust
// Create an instance
let limiter = RateLimiter::memory();
let limiter = RateLimiter::file("storage/rate/");
let limiter = RateLimiter::with_driver(TokenBucketLimiter::new(cache));

// Methods
limiter.attempt("key", 5, 60, || { /* callback */ });  // Result<T, ()>
limiter.too_many_attempts("key", 5);      // bool
limiter.remaining("key", 5);              // u64
limiter.hits("key");                      // u64
limiter.available_in("key");              // u64 (seconds until reset)
limiter.clear("key");                     // reset counter
limiter.hit("key", 60);                   // increment +1
```

| Method | Returns | Laravel Equivalent |
|--------|---------|-------------------|
| `attempt(key, max, decay, fn)` | `Result<T, ()>` | `RateLimiter::attempt()` |
| `too_many_attempts(key, max)` | `bool` | `RateLimiter::tooManyAttempts()` |
| `remaining(key, max)` | `u64` | `RateLimiter::remaining()` |
| `hits(key)` | `u64` | `RateLimiter::attempts()` |
| `available_in(key)` | `u64` | `RateLimiter::availableIn()` |
| `clear(key)` | `()` | `RateLimiter::clear()` |
| `hit(key, decay)` | `()` | `RateLimiter::hit()` |

### Global Functions

```rust
// Auto-initialized with memory cache on first call
RateLimiter::attempt("key", 5, 60, || { ... });
RateLimiter::too_many_attempts("key", 5);
RateLimiter::remaining("key", 5);
RateLimiter::hits("key");
RateLimiter::available_in("key");
RateLimiter::clear("key");
RateLimiter::hit("key", 60);
```

### Custom Initialization

```rust
use viontin::rate_init;

let limiter = RateLimiter::file("storage/rate/");
rate_init(limiter);
```

Also accessible as free functions:

```rust
use viontin::{rate_attempt, too_many_attempts, rate_remaining, rate_hits, available_in, rate_clear, rate_hit};

rate_attempt("key", 5, 60, || { ... });
too_many_attempts("key", 5);
rate_remaining("key", 5);
rate_hits("key");
available_in("key");
rate_clear("key");
rate_hit("key", 60);
```

---

## RateLimiterDriver Trait

Implement for custom backends:

```rust
pub trait RateLimiterDriver: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn attempt(&self, key: &str, max_attempts: u64, decay_seconds: u64) -> bool;
    fn too_many_attempts(&self, key: &str, max_attempts: u64) -> bool;
    fn remaining(&self, key: &str, max_attempts: u64) -> u64;
    fn available_in(&self, key: &str) -> u64;
    fn hits(&self, key: &str) -> u64;
    fn clear(&self, key: &str);
}
```

---

## TokenBucketLimiter

The built-in token bucket implementation:

```rust
use viontin::TokenBucketLimiter;

// With explicit cache driver
let driver = TokenBucketLimiter::new(MemoryCache::new());
let driver = TokenBucketLimiter::new(FileCache::new("storage/rate/"));

// Use with facade
let limiter = RateLimiter::with_driver(driver);
```

Two cache entries per key:

| Cache Key | Value |
|-----------|-------|
| `rate:<key>` | Current hit count |
| `rate:<key>:reset` | Unix timestamp when counter resets |

---

## Complete Example

```rust
use viontin::prelude::*;

fn login_handler(username: &str) -> Response {
    let rate_key = format!("login:{}", username);

    // Laravel-style attempt with callback
    let result = RateLimiter::attempt(&rate_key, 5, 60, || {
        // Only runs if under the rate limit
        authenticate_user(username)
    });

    match result {
        Ok(success) => {
            RateLimiter::clear(&rate_key);
            Response::html("Logged in")
        }
        Err(()) => {
            let retry = RateLimiter::available_in(&rate_key);
            let remaining = RateLimiter::remaining(&rate_key, 5);
            Response::html(
                format!("Rate limited. Retry in {}s ({} remaining)", retry, remaining)
            ).status(StatusCode::TOO_MANY_REQUESTS)
        }
    }
}

fn api_middleware(api_key: &str) -> Result<(), Response> {
    if RateLimiter::too_many_attempts(&format!("api:{}", api_key), 100) {
        Err(Response::html("API rate limit exceeded").status(429))
    } else {
        RateLimiter::hit(&format!("api:{}", api_key), 60);
        Ok(())
    }
}
```
