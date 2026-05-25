> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Testing

**Module:** `viontin_framework::testing` (re-exported from `viontest`)

Testing utilities including fluent expectations, describe/test organization, architecture rule enforcement, and a TUI test runner — all powered by [viontest](viontest) (standalone crate, zero dependencies).

---

## Overview

The testing module is re-exported from **viontest**, a standalone Rust testing crate. You can use it via the framework:

```rust
use viontin::testing::{expect, describe, test, arch};
```

Or directly as a standalone dependency:

```toml
[dependencies]
viontest = { path = "path/to/viontest" }
```

---

## Fluent Expect API

### Basic

```rust
use viontin::testing::expect;

expect(1 + 1).toBe(2);
expect("hello").toContainStr("ell");
expect(5 - 3).toBeGreaterThan(1);
expect(3.14159).toBeCloseTo(3.14, 2);
expect(true).toBeTruthy();
expect(false).toBeFalsy();
```

### Collections

```rust
let v = vec![1, 2, 3];
expect(&v).toContain(&2);
expect(v).toHaveLength(3);
expect::<Vec<i32>>(vec![]).toBeEmpty();
```

### Result

```rust
let ok: Result<i32, String> = Ok(42);
expect(ok).toBeOk().toBeGreaterThan(40);

let err: Result<i32, String> = Err("fail".into());
expect(err).toBeErr();
```

### Option

```rust
let some = Some(42);
expect(some).toBeSome().toBe(42);

let none: Option<i32> = None;
expect(none).toBeNone();
```

### Custom Predicate

```rust
expect(42).toSatisfy(|n| n % 2 == 0, "be even");
```

### Architecture Rules

```rust
use viontin::testing::*;

expect("UserController").toPass(IsPascalCase);
expect("UserController").toPass(EndsWith::new("Controller"));
expect("my_helper").notToPass(IsPascalCase);
expect(checker.check("UserController")).toPassArch();
```

---

## Describe / Test Organization

### Basic

```rust
use viontin::testing::{describe, test, it};

#[test]
fn calculator_tests() {
    describe("Calculator", || {
        test("adds two numbers", || {
            expect(1 + 1).toBe(2);
        });

        it("subtracts", || {
            expect(5 - 3).toBe(2);
        });
    });
}
```

### Hooks

```rust
#[test]
fn with_hooks() {
    describe("Database", || {
        let conn = database::connect();

        beforeEach(|| {
            conn.begin();
        });

        afterEach(|| {
            conn.rollback();
        });

        test("can insert", || {
            conn.insert("users", name: "Alice");
        });

        test("can query", || {
            let users = conn.query("users");
            expect(users).toHaveLength(0);
        });
    });
}
```

### Covers Annotation

```rust
#[test]
fn user_test() {
    covers("models::user");
    covers("services::user");

    test("creates user", || {
        // ...
    });
}
```

### Integration with `#[test]`

The `describe`/`test`/`it` functions work inside Rust's standard `#[test]` functions:

```rust
#[test]
fn suite() {
    describe("Group", || {
        test("case one", || { /* ... */ });
        test("case two", || { /* ... */ });
    });
}
```

---

## Architecture Testing

### Built-in Rules

```rust
use viontin::testing::*;

// Naming conventions
IsPascalCase::new()   // MyStruct, MyTrait
IsCamelCase::new()    // myFunction, myVariable
IsSnakeCase::new()    // my_module, my_function
IsKebabCase::new()    // my-module, my-config

// Dependency rules
DoesNotDependOn::new(&["old_module", "deprecated"])
MustDependOn::new(&["required_module"])
EndsWith::new("Controller")
StartsWith::new("get")
```

### ArchChecker

```rust
let mut checker = ArchChecker::new();
checker = checker
    .add(IsPascalCase)
    .add(DoesNotDependOn::new(&["deprecated"]))
    .add(EndsWith::new("Controller"));

let result = checker.check_all(&["UserController", "old_helper", "AdminController"]);
print_arch_result(&result);
assert!(result.passed(), "Architecture violations found");
```

### Custom ArchRule

```rust
use viontin::testing::*;

#[derive(Debug)]
struct IsNotEmpty;

impl ArchRule for IsNotEmpty {
    fn name(&self) -> &'static str { "is_not_empty" }
    fn check(&self, target: &str) -> ArchFinding {
        let valid = !target.is_empty();
        ArchFinding {
            rule: self.name(),
            target: target.to_string(),
            message: if valid { String::new() } else { "Name must not be empty".into() },
            severity: ArchSeverity::Error,
        }
    }
}
```

---

## Pest-Style `arch()` Builder

### Basic

```rust
use viontin::arch;

arch("UserController")
    .toPascalCase()
    .toEndWith("Controller")
    .assert();
```

### Multiple Targets

```rust
arch("UserController")
    .toPascalCase()
    .toEndWith("Controller")
  .and("PaymentService")
    .toPascalCase()
    .toEndWith("Service")
  .and("user_model")
    .toSnakeCase()
    .assert();
```

### Role Presets

```rust
arch("UserController").toBeController().assert();
arch("PaymentService").toBeService().assert();
arch("User").toBeModel().assert();
arch("AuthMiddleware").toBeMiddleware().assert();
arch("UserRegistered").toBeEvent().assert();
arch("SendWelcomeEmail").toBeJob().assert();
```

| Preset | Checks |
|--------|--------|
| `toBeController()` | PascalCase + ends with "Controller" |
| `toBeService()` | PascalCase + ends with "Service" |
| `toBeModel()` | PascalCase + ends with "Model" |
| `toBeMiddleware()` | PascalCase + ends with "Middleware" |
| `toBeEvent()` | PascalCase + ends with "Event" |
| `toBeJob()` | PascalCase + ends with "Job" |
| `toBeMail()` | PascalCase + ends with "Mail" |
| `toBeNotification()` | PascalCase + ends with "Notification" |
| `toBeProvider()` | PascalCase + ends with "Provider" |

### Negation & Dependencies

```rust
arch("my_helper")
    .notToBePascalCase()
    .toSnakeCase()
    .assert();

arch("use legacy_module::Helper;")
    .notToDependOn(&["legacy_module"])
    .assert();
```

---

## TUI Test Runner

The `TestRunner` and `ConsoleTestReporter` provide industry-standard test output:

```rust
use viontin::testing::{TestRunner, TestResult};

let mut runner = TestRunner::new();
runner.add_result("Unit", TestResult::pass("add", std::time::Duration::from_micros(1200)));
runner.add_result("Unit", TestResult::fail("broken", "expected 2, got 3", std::time::Duration::from_micros(50)));
runner.print();
```

Output:

```
  ✓ add (1.20 ms)
  ✘ broken (0.05 ms)
     → expected 2, got 3

──────────────────────────────────────────────────
 Test Suites: 1 passed | 1 total
 Tests:       1 passed | 1 failed | 2 total
 Duration:    1.25 ms
```

---

## CLI

```bash
# Run all tests
viontin test

# Run with filter
viontin test --filter user_model

# Watch mode
viontin test --watch
```

---

## Integration with CI

```rust
use viontin::testing::*;

#[test]
fn architecture_is_respected() {
    let checker = ArchChecker::new()
        .add(IsPascalCase)
        .add(DoesNotDependOn::new(&["legacy"]))
        .add(EndsWith::new("Controller"));

    let result = checker.check_all(&["UserController", "PaymentService"]);
    print_arch_result(&result);
    assert!(result.passed(), "Architecture violations found");
}
```

Combine with domain boundaries:

```rust
use viontin::{check_domains, DomainViolation};

#[test]
fn no_domain_boundary_violations() {
    let violations = check_domains();
    let errors: Vec<&DomainViolation> = violations.iter()
        .filter(|v| !v.is_warning)
        .collect();
    assert!(errors.is_empty(), "Domain violations: {:?}", errors);
}
```