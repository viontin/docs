# Debugging

**Module:** `viontin_framework::debug`

Developer tools for debugging, profiling, and benchmarking — inspired by Laravel's `dump`/`dd` helpers.

---

## Quick Start

```rust
use viontin::prelude::*;

let user = "Alice";
let count = 42;

dump(&user);   // prints: alloc::string::String("Alice")
dump(&count);  // prints: i64(42)

dd(&user);     // prints and exits
```

---

## Dump & Die

### dump

Print a debug representation of a value to stderr:

```rust
use viontin::prelude::*;

let items = vec![1, 2, 3];
dump(&items);

// Output:
//   Vec<i64>([1, 2, 3])
```

### dd

Dump and exit the process:

```rust
dd(&items);
// Output:
//   Vec<i64>([1, 2, 3])
//   [dd] Execution halted.
```

### dump_many / dd_many

```rust
let a = 1;
let b = "hello";
let c = vec![true, false];

dump_many(&[&a, &b, &c]);
// [0] i64(1)
// [1] alloc::string::String("hello")
// [2] Vec<bool>([true, false])

dd_many(&[&a, &b, &c]); // dump + exit
```

### Macros

The framework also re-exports `dump!` and `dd!` macros with file:line annotation:

```rust
// Not yet available — use dump/dd functions
```

---

## Profiler

Measure execution time of code sections:

```rust
use viontin::prelude::*;

let mut p = Profiler::new();

// Section 1
std::thread::sleep(std::time::Duration::from_millis(50));
p.mark("db-query");

// Section 2
std::thread::sleep(std::time::Duration::from_millis(20));
p.mark("render");

// Section 3
std::thread::sleep(std::time::Duration::from_millis(30));
p.mark("response");

p.report();
```

Output:

```
┌─────────────────────────────────────────────┐
│              Profiler Report                │
├─────────────────────────────────────────────┤
│  db-query                   50.123 ms │
│  render                     20.045 ms │
│  response                   30.067 ms │
├─────────────────────────────────────────────┤
│  Total                     100.235 ms │
└─────────────────────────────────────────────┘
```

| Method | Description |
|--------|-------------|
| `Profiler::new()` | Start profiling |
| `mark(name)` | Record a timing marker |
| `report()` | Print the profiler table |

---

## Benchmark

Measure execution time of a single function:

```rust
use viontin::prelude::*;

// Print only
benchmark("heavy-computation", || {
    let mut x = 0;
    for i in 0..1_000_000 { x += i; }
});

// Print and return value
let result = benchmark_with("parse-data", || {
    "42".parse::<i32>().unwrap()
});
println!("Result: {}", result);
```

Output:

```
[bench] heavy-computation: 2.345 ms
[bench] parse-data: 0.012 μs
```

| Function | Returns | Description |
|----------|---------|-------------|
| `benchmark(name, fn)` | `()` | Measure and print |
| `benchmark_with(name, fn)` | `T` | Measure, print, return result |

---

## Memory Usage

Get current RSS memory usage (Linux only):

```rust
use viontin::prelude::*;

let mem = memory_usage();
println!("{}", mem);
// "VmRSS:   24576 kB"
```

Returns `"N/A"` on non-Linux platforms.

---

## Debug Mode

Check if the application is running in a debug environment:

```rust
use viontin::prelude::*;

if is_debug_mode() {
    // Only runs in local/testing environments
    println!("Debug mode active");
}

// Shorthand for debug-only code
debug_only(|| {
    println!("This only runs in debug mode");
});

// Local-only
when_local(|| {
    println!("This only runs in local environment");
});
```

`is_debug_mode()` returns `true` when:
- `Environment::Local`
- `Environment::Testing`

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    // Debug mode check
    if is_debug_mode() {
        log_info("Debug mode active");
    }

    // Profile a request
    let mut p = Profiler::new();

    let users = benchmark_with("fetch-users", || {
        fetch_users()
    });
    p.mark("fetch");

    let rendered = render_users(&users);
    p.mark("render");

    if is_debug_mode() {
        dump(&users);
        p.report();
    }

    benchmark("process", || {
        // heavy work
    });
}

fn fetch_users() -> Vec<String> {
    vec!["Alice".into(), "Bob".into()]
}

fn render_users(users: &[String]) -> String {
    users.join(", ")
}
```
