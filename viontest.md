# Viontest

**Zero-dependency Rust testing framework** — expect, describe, arch, and a TUI test runner.

Viontest is a standalone crate that provides a complete testing experience for Rust projects. It is the foundation of Viontin's testing capabilities, but works with **any** Rust project.

---

## Why Viontest?

| Feature | Standard Rust | Viontest |
|---------|--------------|----------|
| Assertions | `assert_eq!`, `assert!` | Fluent `expect(value).toBe(2).toBeGreaterThan(1)` |
| Organization | None (`#[test]` only) | `describe`/`test`/`it` with hooks |
| Architecture Testing | None | `ArchRule` + Pest-style `arch()` builder |
| Test Runner Output | Plain text | Colored, hierarchical, Vitest-style |
| Dependencies | None added | **Zero dependencies** |

---

## Installation

### As a Library

```toml
[dependencies]
viontest = { path = "path/to/viontest" }
```

### As a CLI

```bash
cargo install --path path/to/viontest
viontest --help
```

---

## Library API

### expect() — Fluent Assertions

```rust
use viontest::expect;

// Equality
expect(1 + 1).toBe(2);
expect("hello").notToBe("world");

// Comparison
expect(42).toBeGreaterThan(10);
expect(42).toBeLessThanOrEqual(100);

// Float
expect(3.14159).toBeCloseTo(3.14, 2);

// Boolean
expect(true).toBeTruthy();
expect(false).toBeFalsy();

// Collections
let v = vec![1, 2, 3];
expect(&v).toContain(&2);
expect(v).toHaveLength(3);
expect::<Vec<i32>>(vec![]).toBeEmpty();

// Strings
expect("hello world").toStartWith("hello");
expect("hello world").toEndWith("world");
expect("hello world").toContainStr("lo wo");

// Option
let some = Some(42);
expect(some).toBeSome().toBe(42);
let none: Option<i32> = None;
expect(none).toBeNone();

// Result
let ok: Result<i32, String> = Ok(42);
expect(ok).toBeOk().toBeGreaterThan(40);
let err: Result<i32, String> = Err("fail".into());
expect(err).toBeErr();

// Custom predicate
expect(42).toSatisfy(|n| n % 2 == 0, "be even");
```

### describe() / test() / it()

```rust
use viontest::{describe, test, it};

describe("Calculator", || {
    test("adds numbers", || {
        expect(1 + 1).toBe(2);
    });

    it("subtracts numbers", || {
        expect(5 - 3).toBe(2);
    });
});
```

### Hooks

```rust
use viontest::{describe, test, beforeEach, afterEach, beforeAll, afterAll};

describe("Database", || {
    let db = connect();

    beforeAll(|| { db.migrate(); });
    afterAll(|| { db.rollback(); });
    beforeEach(|| { db.begin(); });
    afterEach(|| { db.rollback(); });

    test("inserts", || { /* ... */ });
});
```

### covers()

```rust
use viontest::covers;

covers("modules::user");
test("creates user", || { /* ... */ });
```

### Architecture Testing

```rust
use viontest::*;

// Built-in rules
let checker = ArchChecker::new()
    .add(IsPascalCase)
    .add(IsSnakeCase)
    .add(DoesNotDependOn::new(&["legacy"]))
    .add(EndsWith::new("Controller"));

let result = checker.check_all(&["UserController", "old_helper"]);
print_arch_result(&result);
```

### Pest-Style arch() Builder

```rust
use viontest::arch;

arch("UserController")
    .toPascalCase()
    .toEndWith("Controller")
  .and("payment_service")
    .toSnakeCase()
  .and("AdminController")
    .toBeController()  // shorthand preset
    .assert();
```

| Method | Description |
|--------|-------------|
| `toPascalCase()` | Enforce PascalCase naming |
| `toSnakeCase()` | Enforce snake_case naming |
| `toCamelCase()` | Enforce camelCase naming |
| `toKebabCase()` | Enforce kebab-case naming |
| `toStartWith(prefix)` | Must start with prefix |
| `toEndWith(suffix)` | Must end with suffix |
| `toBeController()` | PascalCase + "Controller" suffix |
| `toBeService()` | PascalCase + "Service" suffix |
| `toBeModel()` | PascalCase + "Model" suffix |
| `toBeMiddleware()` | PascalCase + "Middleware" suffix |
| `toBeEvent()` | PascalCase + "Event" suffix |
| `toBeJob()` | PascalCase + "Job" suffix |
| `toBeMail()` | PascalCase + "Mail" suffix |
| `toBeNotification()` | PascalCase + "Notification" suffix |
| `toBeProvider()` | PascalCase + "Provider" suffix |
| `notToDependOn(list)` | Forbid module dependencies |
| `toDependOn(list)` | Require module dependencies |
| `and(target)` | Chain multiple targets |

---

## CLI

```bash
viontest                    # run all tests
viontest --filter <name>    # run matching tests
viontest --watch            # re-run on file change
viontest --help             # show help
```

The CLI:
1. Runs `cargo test` internally
2. Parses the output (test names, status, failure details)
3. Displays results with colored, hierarchical output

Output example:

```
  ✓ can add (2.34 ms)
  ✓ can subtract (1.02 ms)
  ✘ can divide (0.05 ms)
     → expected Ok, got Err("divide by zero")

──────────────────────────────────────────────────
 Test Suites: 1 passed | 1 failed | 2 total
 Tests:       2 passed | 1 failed | 3 total
 Duration:    3.41 ms
```

---

## Complete Example

```rust
use viontest::{describe, test, expect, arch};

fn main() {
    // Run describe-based tests
    describe("Math", || {
        test("addition", || {
            expect(1 + 1).toBe(2);
            expect(100 + 1).toBeGreaterThan(100);
        });

        test("division", || {
            let result = divide(10, 0);
            expect(result).toBeErr();
        });
    });

    // Run architecture tests
    arch("UserController")
        .toPascalCase()
        .toEndWith("Controller")
      .and("user_service")
        .toSnakeCase()
        .assert();
}

fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 { Err("divide by zero".into()) }
    else { Ok(a / b) }
}
```
