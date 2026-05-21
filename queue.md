> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Queue

**Module:** `viontin_framework::queue`

Queue system for background job processing.

---

## Architecture

```
Queue (facade)
  │
  └── Box<dyn Driver>
        │
        └── SyncQueue  — executes jobs immediately (synchronous)
```

---

## Job Trait

```rust
pub trait Job: Debug + Send + Sync {
    fn handle(self: Box<Self>) -> Result<(), String>;
    fn name(&self) -> &str { std::any::type_name_of_val(self) }
}
```

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct SendWelcomeEmail {
    user_email: String,
}

impl Job for SendWelcomeEmail {
    fn handle(self: Box<Self>) -> Result<(), String> {
        println!("Sending welcome email to {}", self.user_email);
        // actual sending logic
        Ok(())
    }
}
```

---

## Driver Trait

```rust
pub trait Driver: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn push(&self, job: Box<dyn Job>) -> Result<(), String>;
    fn later(&self, delay_secs: u64, job: Box<dyn Job>) -> Result<(), String>;
}
```

---

## SyncQueue

Executes jobs immediately in the same process:

```rust
let driver = SyncQueue;
driver.push(Box::new(SendWelcomeEmail { user_email: "alice@example.com".into() })).unwrap();
```

The `later()` method also runs immediately (no actual delay in sync mode).

---

## Queue Facade

```rust
use viontin::prelude::*;

// With SyncQueue (default)
let queue = Queue::new(SyncQueue);

// Default (also SyncQueue)
let queue = Queue::default();

queue.push(SendWelcomeEmail { user_email: "alice@example.com".into() })?;
queue.later(3600, BackupJob)?;  // delay is ignored in sync mode
queue.driver().name();  // "sync"
```

---

## Integration

The `QueueProvider` registers a `Queue` singleton in the container:

```rust
// Auto-registered by boot()
app.run();
let queue = app.container.resolve::<Queue>();
```

---

## Complete Example

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct ProcessPayment {
    order_id: String,
    amount: i64,
}

impl Job for ProcessPayment {
    fn handle(self: Box<Self>) -> Result<(), String> {
        println!("Processing payment for order {}: {} coins", self.order_id, self.amount);
        Ok(())
    }
}

fn main() {
    let queue = Queue::new(SyncQueue);

    // Queue a job
    queue.push(ProcessPayment {
        order_id: "ORD-789".into(),
        amount: 50000,
    }).unwrap();
}
```
