# Authentication

**Module:** `viontin_framework::auth`  
**Facade:** `Auth`

Authentication system with guard-based architecture, password hashing, and session-aware guards.

---

## Architecture

```
Auth (facade)
  │
  ├── multiple named guards
  ├── default guard routing
  │
  └── Box<dyn AuthGuard>
        │
        ├── BasicGuard     — credential validation via callback
        └── SessionGuard   — wraps any guard with session awareness
```

---

## AuthResult

```rust
pub enum AuthResult {
    Success,
    Failure(String),
}
```

```rust
AuthResult::ok()                    // Success
AuthResult::fail("Invalid password") // Failure("Invalid password")

result.is_success()  // bool
```

---

## Auth Facade

The `Auth` struct manages multiple named guards:

```rust
use viontin::prelude::*;
use std::collections::HashMap;

let mut auth = Auth::new();

// Register a guard
auth.register("web", BasicGuard::new("web", |username, password| {
    // Your authentication logic
    username == "admin" && password == "secret"
}));

// Attempt authentication
let result = auth.attempt(HashMap::from([
    ("username".into(), "admin".into()),
    ("password".into(), "secret".into()),
]));

if result.is_success() {
    println!("Authenticated");
}

// Check current user
if auth.check() {
    let user_id = auth.user();  // Option<String>
}
```

### Methods

| Method | Description |
|--------|-------------|
| `register(name, guard)` | Register a named guard |
| `guard(name)` | Access a specific guard |
| `attempt(credentials)` | Authenticate against the default guard |
| `user()` | Get current user ID from default guard |
| `check()` | Check if authenticated (default guard) |

`attempt` expects a `HashMap<String, String>` with at least `username` (or `email`) and `password` keys.

---

## AuthGuard Trait

```rust
pub trait AuthGuard: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn validate(&self, credentials: &HashMap<String, String>) -> AuthResult;
    fn user_id(&self) -> Option<String>;
    fn set_user(&mut self, id: String);
    fn logout(&mut self);
}
```

| Method | Description |
|--------|-------------|
| `name()` | Guard identifier |
| `validate(credentials)` | Check credentials and return result |
| `user_id()` | Get current authenticated user ID |
| `set_user(id)` | Set the current user |
| `logout()` | Clear the current user |

---

## AuthUser Trait

```rust
pub trait AuthUser {
    fn auth_id(&self) -> String;
    fn auth_password(&self) -> &str;
}
```

Implement on your user model:

```rust
use viontin::prelude::*;

struct User {
    id: String,
    password_hash: String,
}

impl AuthUser for User {
    fn auth_id(&self) -> String {
        self.id.clone()
    }
    fn auth_password(&self) -> &str {
        &self.password_hash
    }
}
```

---

## BasicGuard

Validates credentials using a callback function:

```rust
use viontin::prelude::*;

let guard = BasicGuard::new("web", |username, password| {
    // Lookup user and verify password
    username == "admin" && password == "secret"
});
```

### With Hasher

```rust
use viontin::prelude::*;

let guard = BasicGuard::new("web", |username, password| {
    // Your lookup logic — return true if password matches
    let stored_hash = get_user_hash(username);
    SimpleHasher.verify(password, &stored_hash)
}).with_hasher(SimpleHasher);
```

| Constructor | Description |
|-------------|-------------|
| `BasicGuard::new(name, provider)` | Create with validation callback |
| `.with_hasher(hasher)` | Set a custom hasher |

The `provider` callback receives `(username, password)` and must return `bool`.

### Credential Keys

`BasicGuard::validate()` looks for these keys in the credentials map:
- `username` or `email` — for the user identifier
- `password` — for the password

---

## SessionGuard

Wraps any `AuthGuard` with session-compatible interface:

```rust
use viontin::prelude::*;

let guard = SessionGuard::new(
    BasicGuard::new("web", |u, p| u == "admin" && p == "secret")
);
```

Delegates all calls to the inner guard:

```rust
impl AuthGuard for SessionGuard {
    fn name(&self) -> &str { self.guard.name() }
    fn validate(&self, c: &HashMap<String, String>) -> AuthResult { self.guard.validate(c) }
    fn user_id(&self) -> Option<String> { self.guard.user_id() }
    fn set_user(&mut self, id: String) { self.guard.set_user(id); }
    fn logout(&mut self) { self.guard.logout(); }
}
```

Use with `Session` to persist authentication state across requests:

```rust
use viontin::prelude::*;
use std::collections::HashMap;

fn login(session: &mut Session) {
    let guard = BasicGuard::new("web", |u, p| {
        u == "admin" && p == "secret"
    });

    let result = guard.validate(&HashMap::from([
        ("username".into(), "admin".into()),
        ("password".into(), "secret".into()),
    ]));

    if result.is_success() {
        session.set("auth_guard", "web");
        session.set("user_id", "admin");
    }
}

fn logout(session: &mut Session) {
    session.remove("auth_guard");
    session.remove("user_id");
}
```

---

## Password Hashing

Built-in hasher (development only — NOT cryptographically secure):

```rust
use viontin::prelude::*;

let hasher = SimpleHasher;

// Hash a password
let hash = hasher.hash("my_password");
// Format: "<salt>:<hash>"

// Verify
let valid = hasher.verify("my_password", &hash);
assert!(valid);
```

| Method | Description |
|--------|-------------|
| `hash(value)` | Hash with random salt |
| `verify(value, stored)` | Verify against stored hash |
| `name()` | Returns `"simple"` |

> **Warning:** `SimpleHasher` uses `std::hash::DefaultHasher` which is NOT cryptographically secure. For production, implement the `Hasher` trait with bcrypt, argon2, or PBKDF2 via a Gem.

---

## Hasher Trait

```rust
pub trait Hasher: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn hash(&self, value: &str) -> String;
    fn verify(&self, value: &str, stored: &str) -> bool;
}
```

Custom implementation:

```rust
#[derive(Debug)]
struct BcryptHasher;

impl Hasher for BcryptHasher {
    fn name(&self) -> &str { "bcrypt" }
    fn hash(&self, value: &str) -> String {
        // bcrypt::hash(value, DEFAULT_COST)
        todo!()
    }
    fn verify(&self, value: &str, stored: &str) -> bool {
        // bcrypt::verify(value, stored)
        todo!()
    }
}
```

---

## Complete Example

```rust
use viontin::prelude::*;
use std::collections::HashMap;

fn main() {
    let mut auth = Auth::new();

    // Register guard
    auth.register("web", BasicGuard::new("web", |username, password| {
        // In production: check against database
        username == "alice" && password == "secret123"
    }));

    // Login
    let result = auth.attempt(HashMap::from([
        ("username".into(), "alice".into()),
        ("password".into(), "secret123".into()),
    ]));

    match result {
        AuthResult::Success => {
            println!("Logged in as: {:?}", auth.user());
        }
        AuthResult::Failure(reason) => {
            println!("Login failed: {}", reason);
        }
    }

    // Check auth state
    if auth.check() {
        println!("Authenticated");
    }

    // Access specific guard
    if let Some(guard) = auth.guard("web") {
        println!("Guard: {}", guard.name());
    }
}
```
