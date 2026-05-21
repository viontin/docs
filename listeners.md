# Listeners

**Module:** `viontin_framework::events`

Event listeners and subscribers — define how your application reacts to events.

---

## Listener Trait

```rust
pub trait Listener: Debug + Send + Sync {
    fn listens(&self) -> Vec<&'static str>;
    fn handle(&self, event: &dyn Event);
}
```

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct SendWelcomeEmail;

impl Listener for SendWelcomeEmail {
    fn listens(&self) -> Vec<&'static str> {
        vec!["viontin_framework::events::GenericEvent"]
    }

    fn handle(&self, event: &dyn Event) {
        if let Some(generic) = event.downcast_ref::<GenericEvent>() {
            println!("Handling: {} — {}", generic.name, generic.payload.as_deref().unwrap_or(""));
        }
    }
}
```

Wildcard `"*"` listens to all events:

```rust
fn listens(&self) -> Vec<&'static str> { vec!["*"] }
```

---

## Registering Listeners

```rust
let mut dispatcher = EventDispatcher::new();

dispatcher.listen(
    std::any::type_name_of_val(&MyEvent),
    Box::new(MyListener),
);
```

---

## Subscriber Trait

Subscribers group multiple listener registrations:

```rust
pub trait Subscriber: Debug + Send + Sync {
    fn subscribe(&self, dispatcher: &mut dyn Subscribable);
}
```

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct UserSubscriber;

impl Subscriber for UserSubscriber {
    fn subscribe(&self, dispatcher: &mut dyn Subscribable) {
        dispatcher.listen(
            std::any::type_name_of_val(&GenericEvent),
            Box::new(SendWelcomeEmail),
        );
        dispatcher.listen(
            std::any::type_name_of_val(&GenericEvent),
            Box::new(LogUserAction),
        );
    }
}

// Register:
dispatcher.add_subscriber(UserSubscriber);
```

---

## Subscribable Trait

```rust
pub trait Subscribable: Debug + Send + Sync {
    fn listen(&mut self, event: &'static str, listener: Box<dyn Listener>);
}
```

`EventDispatcher` implements `Subscribable`, so any subscriber can register listeners directly.

---

## Typed Event Example

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct OrderShipped {
    order_id: String,
}

impl Event for OrderShipped {}

#[derive(Debug)]
struct ShipmentNotifier;

impl Listener for ShipmentNotifier {
    fn listens(&self) -> Vec<&'static str> {
        vec![std::any::type_name_of_val(&OrderShipped {
            order_id: String::new(),
        })]
    }

    fn handle(&self, event: &dyn Event) {
        if let Some(order) = event.downcast_ref::<OrderShipped>() {
            println!("Order {} shipped!", order.order_id);
        }
    }
}

fn main() {
    let dispatcher = EventDispatcher::new();
    dispatcher.listen(
        std::any::type_name_of_val(&OrderShipped { order_id: String::new() }),
        Box::new(ShipmentNotifier),
    );

    dispatcher.dispatch(&OrderShipped {
        order_id: "ORD-123".into(),
    });
}
```
