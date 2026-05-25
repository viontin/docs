> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Events

**Module:** `viontin_framework::events`

Pub/sub event system with typed events, wildcard listeners, and subscriber registration.

---

## Quick Start

```rust
use viontin::prelude::*;

// Dispatch a generic event
dispatcher.dispatch(&GenericEvent {
    name: "user.registered".into(),
    payload: Some("user_id:42".into()),
});
```

---

## Event Trait

```rust
pub trait Event: Debug + Send + Sync {}
```

Implement for any type:

```rust
#[derive(Debug)]
struct UserRegistered {
    user_id: i64,
    email: String,
}

impl Event for UserRegistered {}
```

### GenericEvent

For ad-hoc events without defining a type:

```rust
let event = GenericEvent {
    name: "order.shipped".into(),
    payload: Some("order_123".into()),
};
```

---

## EventDispatcher

```rust
pub struct EventDispatcher {
    listeners: HashMap<&'static str, Vec<Box<dyn Listener>>>,
}
```

### Methods

```rust
dispatcher.dispatch(&event);     // dispatch to all matching listeners
dispatcher.add_subscriber(sub);  // register a subscriber
```

Dispatch looks up listeners by the event's `type_name` (from `std::any::type_name_of_val`). Wildcard `"*"` listeners receive all events.

---

## Integration

### Via Service Provider

The `EventsProvider` registers an `EventDispatcher` singleton in the container:

```rust
let app = Application::new();
app.run();

let dispatcher = app.container.resolve::<EventDispatcher>();
```

See [Listeners](listeners) for defining listeners and subscribers.