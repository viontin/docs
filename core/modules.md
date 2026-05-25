> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Module System

**Module:** `viontin_framework::modules`

Organize your application into self-contained modules, each with its own routes, commands, and providers.

---

## Module Trait

```rust
pub trait Module: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn routes(&self, router: Router) -> Router { router }
    fn commands(&self) -> Vec<Box<dyn Command>> { vec![] }
    fn providers(&self) -> Vec<Box<dyn ServiceProvider>> { vec![] }
}
```

---

## Usage

```rust
use viontin::prelude::*;

#[derive(Debug, Clone)]
pub struct Billing;

impl Module for Billing {
    fn name(&self) -> &str { "billing" }

    fn routes(&self, router: Router) -> Router {
        router
            .get("/billing/invoices", Arc::new(list_invoices))
            .post("/billing/pay", Arc::new(pay_invoice))
    }

    fn commands(&self) -> Vec<Box<dyn Command>> {
        vec![Box::new(GenerateInvoiceCommand)]
    }
}
```

## Registration

```rust
boot()
    .module(Billing)
    .module(Shipping)
    .serve(":3000");
```

## Module Groups

Group related modules together:

```rust
boot()
    .module(ModuleGroup::new("sales")
        .add(Billing)
        .add(Shipping)
    )
    .serve(":3000");
```

Module groups are flattened at boot time — no nesting, no hierarchy.

## Scaffolding

```bash
viontin make:module Billing
```

Creates `src/billing/mod.rs` with a `Module` trait implementation template.

## Extraction Path

A module can be extracted into a standalone microservice by:
1. Implementing its routes as a separate HTTP server
2. Registering a `RemoteServiceAdapter` in its place

The `ServiceContract` trait provides the contract boundary:

```rust
use viontin::prelude::*;

// In the monolith:
registry.add(BillingService);

// After extraction:
registry.add(RemoteServiceAdapter::new("billing", "http://billing:8080"));
```

The call site never changes.
