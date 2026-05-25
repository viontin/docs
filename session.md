> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Sessions

**Module:** `viontin_framework::session`  
**Facade:** `Session`

Session management with swappable drivers, flash data, and auto-save on drop.

---

## Architecture

```
Session (facade)
  │
  ├── id generation (auto)
  ├── flash data (auto-cleared on next read)
  ├── auto-save on drop
  │
  └── Box<dyn SessionDriver>
        │
        ├── MemorySession  — in-memory (fast, ephemeral)
        └── FileSession    — file-backed (persistent)
```

---

## Quick Start

```rust
use viontin::prelude::*;

// Create a session
let mut session = Session::memory();

// Store data
session.set("user_id", "42");
session.set("role", "admin");

// Read data
if let Some(user_id) = session.get("user_id") {
    println!("User: {}", user_id);
}

// Peek without marking as read
let role = session.peek("role");

// Check existence
if session.has("user_id") {
    // proceed
}

// Remove
session.remove("temp_data");

// Flash (one-time read data)
session.flash("status", "Changes saved");

// Persist
session.save();
```

> **Note:** `session.save()` is called automatically when the `Session` is dropped.

---

## Session Facade

The `Session` struct wraps a `Box<dyn SessionDriver>` with automatic session ID generation and flash data management.

### Creating a Session

```rust
// In-memory (development)
let session = Session::memory();

// File-backed (production)
let session = Session::file("storage/sessions/");

// With custom driver
use viontin_framework::session::{Session, SessionDriver, MemorySession};
let session = Session::with_driver(MemorySession::new());
```

Each session gets a unique ID generated from subsecond timestamps:

```
sess_1a2b3c4f
```

### Methods

```rust
fn get(&mut self, key: &str) -> Option<String>        // read (marks flash as consumed)
fn peek(&self, key: &str) -> Option<&str>              // read (no side effects)
fn set(&mut self, key: impl Into<String>, value: impl Into<String>)
fn has(&self, key: &str) -> bool
fn remove(&mut self, key: &str) -> Option<String>
fn flash(&mut self, key: impl Into<String>, value: impl Into<String>)
fn save(&self)                                          // persist to driver
fn destroy(&mut self)                                   // clear + remove from driver
```

| Method | Description |
|--------|-------------|
| `get(key)` | Read a value; marks `_`-prefixed keys as consumed |
| `peek(key)` | Read without marking consumed |
| `set(key, value)` | Store a value |
| `has(key)` | Check if key exists |
| `remove(key)` | Delete a key |
| `flash(key, value)` | Store one-time data (prefixed with `_`) |
| `save()` | Persist to driver |
| `destroy()` | Clear all data and remove session |

### Session ID

The session ID is generated internally and currently not exposed via a public getter.

### Auto-Save

`Session` implements `Drop`, so `save()` is called automatically when the session goes out of scope:

```rust
{
    let mut session = Session::memory();
    session.set("key", "value");
    // session.save() called here implicitly
}
```

---

## Flash Data

Flash data is stored with a `_` prefix and automatically removed after the first `get()` call:

```rust
fn handle_request() {
    let mut session = Session::memory();

    // Store flash data
    session.flash("status", "Profile updated");
    session.flash("error", "Invalid email");

    // On the next request:
    let msg = session.get("_status");
    // msg is returned, and "_status" is marked for deletion

    // When save() is called, marked keys are cleaned up
    session.save();
}
```

This pattern is useful for "flash messages" that should survive a redirect but be shown only once.

---

## SessionDriver Trait

Implement for custom session backends:

```rust
pub trait SessionDriver: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn read(&self, id: &str) -> HashMap<String, String>;
    fn write(&self, id: &str, data: &HashMap<String, String>);
    fn destroy(&self, id: &str);
    fn gc(&self, max_lifetime: u64);
}
```

| Method | Description |
|--------|-------------|
| `read(id)` | Load session data by ID |
| `write(id, data)` | Persist session data |
| `destroy(id)` | Delete session |
| `gc(max_lifetime)` | Garbage collect expired sessions |

---

## Built-in Drivers

### MemorySession

In-memory storage using `HashMap` behind a `Mutex`.

```rust
Session::memory()
```

**Characteristics:**
- Fastest option
- Data lost on process restart
- No garbage collection needed
- Best for development and single-process apps

### FileSession

Persists each session as a file on disk.

```rust
Session::file("storage/sessions/")
```

File format:

```
<expiry_timestamp>
key1=value1
key2=value2
```

**Characteristics:**
- Persistent across restarts
- 2-day TTL enforced
- Garbage collection via `gc(max_lifetime)`
- Best for single-server production deployments

#### Garbage Collection

```rust
use viontin_framework::session::SessionDriver;

let driver = FileSession::new("storage/sessions/");
driver.gc(86400 * 2);  // remove sessions older than 2 days
```

---

## Custom Driver

```rust
use viontin_framework::session::{Session, SessionDriver};
use std::collections::HashMap;

#[derive(Debug)]
struct RedisSession;
// or DatabaseSession, CookieSession, etc.

impl SessionDriver for RedisSession {
    fn name(&self) -> &str { "redis" }
    fn read(&self, id: &str) -> HashMap<String, String> { /* ... */ }
    fn write(&self, id: &str, data: &HashMap<String, String>) { /* ... */ }
    fn destroy(&self, id: &str) { /* ... */ }
    fn gc(&self, max_lifetime: u64) { /* ... */ }
}

let session = Session::with_driver(RedisSession);
```

---

## Integration with HTTP

Sessions can be tied to HTTP cookies:

```rust
use viontin::prelude::*;

fn login(req: Request) -> Response {
    let mut session = Session::memory();

    // Authenticate user
    session.set("user_id", "42");
    session.set("role", "admin");
    session.flash("welcome", "Welcome back!");

    session.save();  // persist

    // Note: session.id() is internal — use a custom session ID for cookies
    // let sid = "session_...";
    Response::html("<h1>Logged in</h1>")
        .cookie("session_id", "custom_session_id")
}
```

On subsequent requests, load the session from the cookie:

```rust
fn dashboard(req: Request) -> Response {
    let sid = req.cookie("session_id").unwrap_or_default();
    let driver = FileSession::new("storage/sessions/");
    let mut data = driver.read(&sid);

    let user_id = data.get("user_id").cloned();
    let role = data.get("role").cloned();

    // ... build response
}
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    let mut session = Session::memory();

    // Set session data
    session.set("username", "alice");
    session.set("theme", "dark");

    // Flash a one-time message
    session.flash("notification", "Settings saved");

    // Check and read
    if session.has("username") {
        let username = session.get("username").unwrap();
        println!("Welcome back, {}", username);
    }

    // Peek without consuming flash
    if let Some(notif) = session.peek("_notification") {
        println!("Notification: {}", notif);
    }

    // Remove old data
    session.remove("temp_token");

    // Persist (optional — auto-saved on drop)
    session.save();
}
```