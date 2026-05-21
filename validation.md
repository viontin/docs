> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Validation

**Module:** `viontin_framework::validator`

Input validation framework with composable validators, severity levels, and grouped validation.

---

## Quick Start

```rust
use viontin::prelude::*;

let mut outcome = Outcome::new();
outcome.error("E001", "Name is required");
outcome.warning("W001", "Email is deprecated");

if outcome.has_errors() {
    for err in outcome.errors() {
        println!("{}: {}", err.code, err.message);
    }
}
```

---

## Severity

```rust
pub enum Severity {
    Error,
    Warning,
    Info,
}
```

---

## Finding

```rust
pub struct Finding {
    pub severity: Severity,
    pub code: &'static str,
    pub message: String,
    pub location: Option<String>,
}

Finding {
    severity: Severity::Error,
    code: "E001",
    message: "Name is required".into(),
    location: Some("src/forms/user.rs:42".into()),
}
```

---

## Outcome

```rust
pub struct Outcome {
    pub findings: Vec<Finding>,
}
```

### Methods

```rust
let mut outcome = Outcome::new();
outcome.error("E001", "failed");
outcome.warning("W001", "deprecated");
outcome.info("I001", "note");
outcome.add(finding);

outcome.has_errors();   // bool
outcome.errors();       // Vec<&Finding> (filtered)
outcome.is_empty();     // bool
outcome.merge(other);   // combine findings
```

---

## Validator Trait

```rust
pub trait Validator: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn validate(&self, ctx: &Context) -> Outcome;
}
```

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct NameValidator;

impl Validator for NameValidator {
    fn name(&self) -> &str { "name" }

    fn validate(&self, ctx: &Context) -> Outcome {
        let mut result = Outcome::new();
        // validate based on context
        result
    }
}
```

---

## Context

```rust
pub struct Context {
    pub project_root: Option<String>,
    pub source_files: Vec<String>,
    pub config: Option<String>,
}
```

---

## ValidatorGroup

Run multiple validators and collect all outcomes:

```rust
use viontin::prelude::*;

let group = ValidatorGroup::new()
    .add(NameValidator)
    .add(EmailValidator)
    .add(CustomValidator);

let ctx = Context::default();
let outcome = group.validate_all(&ctx);

if outcome.has_errors() {
    for err in outcome.errors() {
        println!("[{}] {}", err.code, err.message);
    }
}
```

---

## Complete Example

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct RequiredFields {
    fields: Vec<String>,
}

impl Validator for RequiredFields {
    fn name(&self) -> &str { "required_fields" }

    fn validate(&self, ctx: &Context) -> Outcome {
        let mut result = Outcome::new();
        for field in &self.fields {
            if !ctx.source_files.iter().any(|f| f.contains(field)) {
                result.warning("M001", &format!("Missing field: {}", field));
            }
        }
        result
    }
}

fn main() {
    let ctx = Context {
        project_root: Some("./".into()),
        source_files: vec!["name".into(), "email".into()],
        config: None,
    };

    let outcome = ValidatorGroup::new()
        .add(RequiredFields { fields: vec!["name".into(), "email".into(), "age".into()] })
        .validate_all(&ctx);

    if outcome.has_errors() {
        eprintln!("Validation failed!");
    }
}
```
