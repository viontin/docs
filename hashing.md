# Hashing

**Module:** `viontin_framework::support::hash`

Password hashing and verification, content hashing, and random token generation.

---

## Hasher Trait

```rust
pub trait Hasher: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn hash(&self, value: &str) -> String;
    fn verify(&self, value: &str, hash: &str) -> bool;
    fn needs_rehash(&self, _hash: &str) -> bool { false }
}
```

| Method | Returns | Description |
|--------|---------|-------------|
| `hash(value)` | `String` | Hash a value with salt |
| `verify(value, hash)` | `bool` | Verify against stored hash |
| `needs_rehash(hash)` | `bool` | Check if hash needs updating |
| `name()` | `&str` | Algorithm identifier |

---

## SimpleHasher

Development-only hasher using `std::hash::DefaultHasher`:

```rust
use viontin::prelude::*;

let hasher = SimpleHasher;

// Hash a password
let hash = hasher.hash("my_password");
// Format: "<salt>:<hex_hash>"

// Verify
assert!(hasher.verify("my_password", &hash));
assert!(!hasher.verify("wrong_password", &hash));
```

```
hash output: "1a2b3c4f:abcdef1234567890"
              └─salt─┘ └───hash────┘
```

> **Warning:** `SimpleHasher` uses `std::hash::DefaultHasher` which is **NOT cryptographically secure**. It is suitable for development and testing only. For production, implement `Hasher` with bcrypt, argon2, or PBKDF2.

---

## Utility Functions

### hex_digest

Quick hex hash of a string:

```rust
use viontin::prelude::*;

let digest = hex_digest("hello");
// "5e7d28c2b79bba02"
```

### quick_hash

Hash any `Hash`-able value to a `u64`:

```rust
use viontin::prelude::*;

let num: u64 = quick_hash(&"hello");
// 1638552786...
```

### random_token

Generate a random alphanumeric token:

```rust
use viontin::prelude::*;

let token = random_token(32);
// "a3x9k2m7..." (32 characters, a-z + 0-9)
```

---

## Custom Hasher

For production, implement `Hasher` with a secure algorithm:

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct BcryptHasher;

impl Hasher for BcryptHasher {
    fn name(&self) -> &str { "bcrypt" }

    fn hash(&self, value: &str) -> String {
        // bcrypt::hash(value, DEFAULT_COST)
        todo!("Implement with bcrypt crate")
    }

    fn verify(&self, value: &str, stored_hash: &str) -> bool {
        // bcrypt::verify(value, stored_hash)
        todo!("Implement with bcrypt crate")
    }

    fn needs_rehash(&self, hash: &str) -> bool {
        // Check if hash uses outdated cost factor
        false
    }
}

// Usage
let hasher = BcryptHasher;
let hash = hasher.hash("user_password");
assert!(hasher.verify("user_password", &hash));
```

---

## Integration with Auth

```rust
use viontin::prelude::*;
use std::collections::HashMap;

let guard = BasicGuard::new("web", |username, password| {
    // In production: fetch user from database
    let stored_hash = get_password_hash(username);
    SimpleHasher.verify(password, &stored_hash)
}).with_hasher(SimpleHasher);

let mut auth = Auth::new();
auth.register("web", guard);

let result = auth.attempt(HashMap::from([
    ("username".into(), "alice".into()),
    ("password".into(), "secret123".into()),
]));
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    // Hash a password
    let password = "correct-horse-battery-staple";
    let hash = SimpleHasher.hash(password);
    println!("Hash: {}", hash);

    // Verify
    assert!(SimpleHasher.verify(password, &hash));
    assert!(!SimpleHasher.verify("wrong password", &hash));

    // Generate tokens
    let api_key = random_token(48);
    println!("API Key: {}", api_key);

    let reset_token = random_token(32);
    println!("Reset Token: {}", reset_token);

    // Quick content hash
    let content = "file contents";
    let digest = hex_digest(content);
    println!("Digest: {}", digest);
}
```
