> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Building Terminal Applications

---

Terminal applications come in two flavors: **CLI commands** (parse, execute, output) and **interactive prompts** (ask, validate, respond). Viontin supports both through the `cli` and `tui` modules.

---

## 1. CLI Commands

The CLI system uses a `Command` trait with Laravel-inspired signature parsing.

### Defining a Command

```rust
use viontin::prelude::*;

struct GreetCommand;

impl Command for GreetCommand {
    fn signature(&self) -> &Signature {
        &Signature::new("greet {name} {--shout} {--times=1}")
    }

    fn description(&self) -> &str {
        "Greet someone with optional shouting"
    }

    fn handle(&self, input: Input, output: Output) -> ExitCode {
        let name = input.argument("name").unwrap();
        let times: usize = input.option("times").unwrap_or("1").parse().unwrap_or(1);
        let shout = input.has_flag("shout");

        for _ in 0..times {
            let mut msg = format!("Hello, {}!", name);
            if shout {
                msg = msg.to_uppercase();
            }
            output.line(&msg);
        }

        ExitCode::SUCCESS
    }
}

fn main() {
    boot()
        .command(GreetCommand)
        .run_with(|_ctx| {});
}
```

```bash
cargo run -- greet World --shout --times=3
# HELLO, WORLD!
# HELLO, WORLD!
# HELLO, WORLD!
```

### Signature Syntax

```
command:name {argument} {optional?} {array*} {--flag} {--option=default}
```

| Pattern | Type | Description |
|---------|------|-------------|
| `{name}` | Required | Positional argument |
| `{name?}` | Optional | Optional positional |
| `{name*}` | Array | Zero or more values |
| `{--flag}` | Boolean | True if present |
| `{--name=}` | Required | Option with value |
| `{--name=val}` | Optional | Option with default |

### Input API

```rust
fn handle(&self, input: Input, output: Output) -> ExitCode {
    input.argument("name");          // Option<String>
    input.arguments();               // Vec<String>
    input.has_flag("force");         // bool
    input.option("path");            // Option<String>
}
```

### Output API

```rust
fn handle(&self, _input: Input, output: Output) -> ExitCode {
    output.line("Normal text");
    output.info("Info");
    output.warn("Warning");
    output.error("Error");
    output.success("Success");

    output.table(&[
        ("Name", "Age"),
        ("Alice", "30"),
        ("Bob", "25"),
    ]);

    ExitCode::SUCCESS   // 0
    ExitCode::FAILURE   // 1
    ExitCode::INVALID   // 2
}
```

---

## 2. Spinner & Progress Bar

```rust
fn handle(&self, _input: Input, output: Output) -> ExitCode {
    // Indeterminate progress
    let spin = output.spinner("Processing...");
    std::thread::sleep(std::time::Duration::from_secs(2));
    spin.finish("Done!");

    // Determinate progress
    let bar = output.progress_bar(100);
    for i in 0..100 {
        std::thread::sleep(std::time::Duration::from_millis(20));
        bar.advance(1);
    }
    bar.finish();

    ExitCode::SUCCESS
}
```

---

## 3. Multiple Commands

```rust
fn main() {
    boot()
        .command(BackupCommand)
        .command(RestoreCommand)
        .command(HealthCommand)
        .run_with(|_ctx| {
            println!("Available commands:");
            println!("  backup  [--force]");
            println!("  restore <name>");
            println!("  health");
        });
}
```

Dispatch:

```bash
cargo run -- backup --force
cargo run -- restore snapshot-latest
cargo run -- health
```

---

## 4. Interactive Prompts

For when you need input from the user during a terminal session:

```rust
use viontin_tui::prompts::{text, select, confirm, password};
use viontin_tui::styling::*;
}

fn interactive_setup() {
    println!("{}", bold("Setup Wizard\n"));

    let name = text("Project name:")
        .with_placeholder("my-app")
        .with_validation(|s| {
            if s.is_empty() {
                Err("Name is required".into())
            } else {
                Ok(())
            }
        })
        .prompt()
        .unwrap();

    let template = select("Template:")
        .choices(vec!["web", "cli", "library"])
        .default("web")
        .prompt()
        .unwrap();

    let git = confirm("Initialize git?")
        .default(true)
        .prompt()
        .unwrap();

    let pwd = password("Database password:")
        .prompt()
        .unwrap();

    println!("\n{}", green("Done!"));
    println!("  Name:     {}", bold(&name));
    println!("  Template: {}", template);
    println!("  Git:      {}", git);
}
```

---

## 5. ANSI Styling

Style terminal output without a prompt:

```rust
use viontin_tui::styling::*;

println!("{}", bold("Bold"));
println!("{}", dim("Dim"));
println!("{}", italic("Italic"));
println!("{}", underline("Underline"));
println!("{}", red("Red"));
println!("{}", green("Green"));
println!("{}", bg_blue("Blue background"));
println!("{}", bold(red("Bold red")));
println!("{}", color_256(82, "Bright green"));
```

---

## 6. Library Mode (No Command Dispatch)

When run without arguments, the closure executes instead:

```rust
fn main() {
    boot()
        .command(GreetCommand)
        .run_with(|_ctx| {
            // No CLI args provided — run default behavior
            println!("Usage: cargo run -- greet <name>");
        });
}
```

Useful for daemons, background workers, or interactive loops that also expose CLI commands.

---

## 7. Full Example

```rust
use viontin::prelude::*;

struct SearchCommand;

impl Command for SearchCommand {
    fn signature(&self) -> &Signature {
        &Signature::new("search {query} {--limit=10} {--json}")
    }

    fn description(&self) -> &str {
        "Search for packages"
    }

    fn handle(&self, input: Input, output: Output) -> ExitCode {
        let query = input.argument("query").unwrap();
        let limit = input.option("limit").unwrap_or("10".into());
        let json = input.has_flag("json");

        let results = perform_search(&query, &limit);

        if json {
            output.line(&format!("{}", serde_json::to_string(&results).unwrap()));
        } else {
            for r in &results {
                output.line(&format!("  {} — {}", r.name, r.description));
            }
            output.info(&format!("{} result(s)", results.len()));
        }

        ExitCode::SUCCESS
    }
}

fn perform_search(query: &str, limit: &str) -> Vec<SearchResult> {
    // ... search logic ...
    vec![]
}

fn main() {
    boot()
        .command(SearchCommand)
        .run_with(|_ctx| {
            println!("Run `cargo run -- search <query>`");
        });
}
```
