# Domain-Driven Design

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Requires `domain` feature.

**Module:** `viontin_framework::domain` (gated behind `domain` feature)  
**Proc-macros:** `viontin-macros` (`#[domain]`, `#[domain_event]`)

Viontin provides built-in support for Domain-Driven Design — bounded contexts, domain events, aggregate roots, and repositories.

---

## Enabling

```toml
[dependencies]
viontin = { features = ["domain"] }
```

---

## Domain

A domain is a bounded context. Each domain declares which other domains it may depend on:

### Via Declarative Macro

```rust
use viontin::domain;

domain!(billing, allows: [order, payment]);
domain!(order, allows: [payment]);
domain!(payment, allows: []);
```

### Via Proc-Macro Attribute

```rust
use viontin::macros::domain;

#[domain(name = "billing", allows = [order, payment])]
mod billing {
    pub struct Invoice;
    pub fn process() -> String { "processed".into() }
}
```

### Domain Struct

```rust
use viontin::Domain;

let billing = Domain::new("billing")
    .allows(&["order", "payment"])
    .provides(&["invoice", "payment"]);
```

### Registration

```rust
use viontin::register_domain;

register_domain(billing);
```

### Querying

```rust
use viontin::{domains, find_domain, domain_is_allowed};

let all = domains();
let billing = find_domain("billing");
let allowed = domain_is_allowed("billing", "order");
```

---

## DomainServiceProvider

The `DomainServiceProvider` is the entry point for registering domains and their listeners:

```rust
use viontin::boot;
use viontin::domain::{DomainServiceProvider, DomainConfig, Domain};

fn main() {
    boot()
        .provider(DomainServiceProvider::new()
            .domain(DomainConfig::new(Domain::new("billing")
                .allows(&["order"])
            ))
            .domain(DomainConfig::new(Domain::new("order")
                .allows(&["payment"])
            ))
            .domain(DomainConfig::new(Domain::new("payment")
                .allows(&[])
            ))
        )
        .serve(":3000");
}
```

---

## Domain Events

Domain events extend the basic event system. A `DomainEvent` is also an `Event`, so it works with the standard `EventDispatcher`.

### Via Proc-Macro

```rust
use viontin::macros::domain_event;

#[domain_event("billing")]
struct InvoicePaid {
    invoice_id: String,
    amount: f64,
}

// Generates:
// - impl Event for InvoicePaid
// - impl DomainEvent for InvoicePaid { domain() -> "billing"; event_name() -> "InvoicePaid" }
```

### Manual Implementation

```rust
use viontin::{DomainEvent, Event};

#[derive(Debug, Clone)]
struct OrderShipped {
    order_id: String,
}

impl Event for OrderShipped {}

impl DomainEvent for OrderShipped {
    fn domain(&self) -> &str { "order" }
    fn event_name(&self) -> &str { "order.shipped" }
}
```

### GenericDomainEvent

```rust
use viontin::domain::GenericDomainEvent;

let event = GenericDomainEvent {
    domain: "billing".into(),
    name: "invoice.paid".into(),
    payload: Some("invoice_42".into()),
};
```

---

## Domain Listeners

Domain listeners work with the standard `EventDispatcher`:

```rust
use viontin::domain::DomainListener;
use viontin::events::EventDispatcher;

#[derive(Debug)]
struct InvoiceLogger;

impl DomainListener for InvoiceLogger {
    fn domain(&self) -> &str { "billing" }

    fn handle_domain(&self, event: &dyn DomainEvent) {
        println!("[{}] {}: {:?}", event.domain(), event.event_name(), event);
    }
}

let mut dispatcher = EventDispatcher::new();
```

---

## AggregateRoot

An aggregate root guarantees consistency boundaries within a domain:

```rust
use viontin::domain::{AggregateRoot, DomainEvent};

#[derive(Debug)]
struct Order {
    id: String,
    items: Vec<String>,
    events: Vec<Box<dyn DomainEvent>>,
}

impl AggregateRoot for Order {
    fn domain(&self) -> &str { "order" }
    fn id(&self) -> &str { &self.id }
    fn events(&self) -> Vec<Box<dyn DomainEvent>> { Vec::new() }
    fn clear_events(&mut self) { self.events.clear() }
}
```

---

## Repository

Repository provides data access abstraction within a domain:

```rust
use viontin::domain::Repository;

#[derive(Debug)]
struct OrderRepository;

impl Repository<Order> for OrderRepository {
    fn domain(&self) -> &str { "order" }

    fn save(&self, order: &Order) -> Result<()> {
        println!("Saving order {} to database", order.id());
        Ok(())
    }

    fn find_by_id(&self, id: &str) -> Result<Option<Order>, String> {
        Ok(Some(Order {
            id: id.into(),
            items: vec![],
            events: vec![],
        }))
    }

    fn delete(&self, order: &Order) -> Result<()> {
        println!("Deleting order {}", order.id());
        Ok(())
    }
}
```

---

## Boundary Checking

```bash
viontin check --arch
```

Scans `src/domain/*/` for cross-domain import violations:
- **Error** (✘): Import from domain NOT in `allows` list
- **Warning** (⚠): Import from domain IN `allows` list (visible for review)

### Programmatic API

```rust
use viontin::{check_domains, DomainViolation};

let violations = check_domains();
for v in &violations {
    println!("{} → {} in {}", v.from_domain, v.to_domain, v.source_file);
}
```

### Architecture Testing Integration

```rust
use viontin::testing::{arch, IsPascalCase};

#[test]
fn domain_controllers() {
    arch("BillingController")
        .toBeController()
      .and("OrderController")
        .toBeController()
        .assert();
}

#[test]
fn no_domain_violations() {
    let violations = check_domains();
    assert!(violations.is_empty(), "Domain violations found");
}
```

---

## Complete Example

```toml
[dependencies]
viontin = { features = ["domain"] }
```

```rust
use viontin::macros::domain;
use viontin::domain::{DomainServiceProvider, DomainConfig, Domain};
use viontin::boot;

#[domain(name = "billing", allows = [order])]
mod billing {
    pub struct Invoice;
}

fn main() {
    boot()
        .provider(DomainServiceProvider::new()
            .domain(DomainConfig::new(Domain::new("billing")
                .allows(&["order"])
            ))
            .domain(DomainConfig::new(Domain::new("order")
                .allows(&["billing"])
            ))
        )
        .serve(":3000");
}
```
