> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Logging

**Module:** `viontin_framework::log`  
**Facade:** `Logger`

Multi-channel logging with severity levels, structured output, and a global logger.

---

## Quick Start

```rust
use viontin::prelude::*;

log_emergency("System is shutting down");
log_alert("Database connection pool exhausted");
log_critical("Disk space critically low");
log_error("Failed to connect to database");
log_warning("Disk space low");
log_notice("Configuration reloaded");
log_info("Application started");
log_debug("Query: SELECT * FROM users");
```

Output (structured format):

```
[2025-01-15T10:30:00Z] app.EMERGENCY: System shutting down
[2025-01-15T10:30:00Z] app.ALERT: Connection pool exhausted
[2025-01-15T10:30:00Z] app.CRITICAL: Disk space critically low
[2025-01-15T10:30:00Z] app.ERROR: Failed to connect to database
[2025-01-15T10:30:00Z] app.WARNING: Disk space low
[2025-01-15T10:30:00Z] app.NOTICE: Configuration reloaded
[2025-01-15T10:30:00Z] app.INFO: Application started
[2025-01-15T10:30:00Z] app.DEBUG: Query: SELECT * FROM users
```

---

## Logger

### Creating a Logger

```rust
use viontin::prelude::*;

// Default logger (stdout, auto-detect min level)
let logger = Logger::new()
    .add_channel(StdoutLog::new(Level::Debug));

// With environment-aware level
let logger = default_logger();
// Local → Debug level
// Production → Warning level
```

### Methods

```rust
logger.emergency("message");
logger.alert("message");
logger.critical("message");
logger.error("message");
logger.warning("message");
logger.notice("message");
logger.info("message");
logger.debug("message");
logger.log(LogEntry { /* ... */ });
```

---

## Levels

```rust
pub enum Level {
    Emergency,  // 0 (highest)
    Alert,      // 1
    Critical,   // 2
    Error,      // 3
    Warning,    // 4
    Notice,     // 5
    Info,       // 6
    Debug,      // 7 (lowest)
}
```

```rust
Level::Info.as_str();   // "INFO"
Level::Debug.as_str();  // "DEBUG"
```

Levels are ordered — `StdoutLog` with `min_level: Warning` will output Warning, Error, Critical, Alert, and Emergency but suppress Notice, Info, and Debug.

---

## LogEntry

```rust
pub struct LogEntry {
    pub level: Level,
    pub message: String,
    pub channel: String,
    pub context: Vec<(String, String)>,
    pub timestamp: String,
}
```

```rust
use viontin::prelude::*;

logger.log(LogEntry {
    level: Level::Error,
    message: "Database query failed".into(),
    channel: "sql".into(),
    context: vec![
        ("query".into(), "SELECT * FROM users".into()),
        ("duration".into(), "12.3s".into()),
    ],
    timestamp: String::new(), // auto-filled by channel
});
```

Structured output with context:

```
[2025-01-15T10:30:00Z] sql.ERROR: Database query failed {"query": "SELECT * FROM users", "duration": "12.3s"}
```

---

## LogChannel Trait

```rust
pub trait LogChannel: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn write(&self, entry: &LogEntry);
    fn is_enabled_for(&self, _level: Level) -> bool { true }
}
```

---

## StdoutLog

Built-in channel that writes to stdout/stderr:

```rust
use viontin::prelude::*;

// Structured format (default)
let channel = StdoutLog::new(Level::Debug)
    .with_format(LogFormat::Structured);

// Simple format
let channel = StdoutLog::new(Level::Warning)
    .with_format(LogFormat::Simple);
```

### Formats

**Structured:**
```
[timestamp] channel.LEVEL: message {"key": "val"}
```

**Simple:**
```
[ERROR] message
```

### Level Filtering

```rust
let channel = StdoutLog::new(Level::Warning);
// Only Warning, Error, Critical, Alert, Emergency pass through
channel.is_enabled_for(Level::Info);    // false
channel.is_enabled_for(Level::Error);   // true
```

---

## Global Logger

```rust
// Initialize once at startup
init_logger(Logger::new()
    .add_channel(StdoutLog::new(Level::Debug))
);

// Use anywhere — all levels available
log_emergency("system failure");
log_alert("connection pool depleted");
log_critical("disk space critical");
log_error("failed");
log_warning("warning");
log_notice("config reloaded");
log_info("started");
log_debug("debug info");
```

### Initialization via Service Provider

```rust
use viontin::prelude::*;

// The LogProvider is registered by default in boot()
// It calls init_logger(default_logger()) automatically

// To customize:
boot()
    .without_provider("log")
    .provider(CustomLogProvider)
    // ...
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    // Configure logger
    let logger = Logger::new()
        .add_channel(
            StdoutLog::new(Level::Debug)
                .with_format(LogFormat::Structured)
        );

    init_logger(logger);

    // Application
    log_info("Application booting");

    match connect_database() {
        Ok(()) => log_info("Database connected"),
        Err(e) => log_critical(format!("Database connection failed: {}", e)),
    }

    log_debug("Configuration loaded");
    log_warning("Running in development mode");
    log_notice("Using memory cache driver");
}

fn connect_database() -> Result<(), String> {
    // ...
    Ok(())
}
```