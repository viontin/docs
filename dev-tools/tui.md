> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# TUI Toolkit

**Crate:** `viontin-tui`  
**Module:** `viontin_tui`

The TUI toolkit provides building blocks for rich terminal applications — interactive prompts, ANSI text styling, and input validation — all built on top of the framework's CLI abstractions.

---

## Features

| Feature | Default | Description |
|---------|---------|-------------|
| `prompts` | Enabled | Interactive prompts (requires `crossterm`) |

```toml
[dependencies]
# All features (default)
viontin-tui = { path = "../../products/tui" }

# CLI-only (no prompts)
viontin-tui = { path = "../../products/tui", default-features = false }
```

---

## Module Overview

```
viontin_tui
├── Kernel          # (re-export from framework) command dispatcher
├── Command         # (re-export) trait for CLI commands
├── Input           # (re-export) parsed CLI arguments
├── Output          # (re-export) styled terminal output
├── ExitCode        # (re-export) exit status codes
│
├── styling         # ANSI text styling
│   ├── bold       │  dim        │  italic
│   ├── underline  │  strikethrough
│   ├── red/green/yellow/blue/magenta/cyan/white/grey
│   └── bg_red/bg_green/bg_blue/bg_cyan
│
├── prompts         # (feature = "prompts")
│   ├── text       │  select     │  confirm    │  password
│   └── crossterm  (re-export for custom prompts)
│
└── validator       # CLI signature & input validation
    ├── validate_signature()
    ├── validate_input()
    ├── validate_command_name()
    ├── Finding     │  Outcome    │  Severity
    └── codes: C001—C022
```

---

## Styling

The `styling` module wraps ANSI escape codes for terminal output formatting.

### Text Styles

```rust
use viontin_tui::styling::*;

println!("{}", bold("Bold text"));
println!("{}", dim("Dim / muted text"));
println!("{}", italic("Italic text"));
println!("{}", underline("Underlined text"));
println!("{}", strikethrough("Strikethrough text"));
```

### Foreground Colors

```rust
println!("{}", red("red"));
println!("{}", green("green"));
println!("{}", yellow("yellow"));
println!("{}", blue("blue"));
println!("{}", magenta("magenta"));
println!("{}", cyan("cyan"));
println!("{}", white("white"));
println!("{}", grey("grey"));
```

### Background Colors

```rust
println!("{}", bg_red("red bg"));
println!("{}", bg_green("green bg"));
println!("{}", bg_blue("blue bg"));
println!("{}", bg_cyan("cyan bg"));
```

### Composition

```rust
println!("{}", bold(red("Bold red")));
println!("{}", bold(underline("Bold underlined")));
println!("{}", green(bold("Green bold")));
```

### Custom ANSI Codes

```rust
use viontin_tui::styling::style;

// ANSI code 93 = bright yellow foreground
println!("{}", style("Bright yellow", "93"));

// ANSI code 1;31 = bold red
println!("{}", style("Bold red", "1;31"));
```

---

## Interactive Prompts

Requires the `prompts` feature (enabled by default). All prompts use `crossterm` for raw terminal input.

### `text` — Text Input

```rust
use viontin_tui::prompts::text;

let name = text("What is your name?")
    .placeholder("e.g. Alice")
    .default("Guest")
    .required(true)
    .validate(|s| {
        if s.len() < 2 {
            Err("Name must be at least 2 characters".into())
        } else {
            Ok(())
        }
    })
    .prompt();  // Result<String, String>
```

| Method | Description |
|--------|-------------|
| `.placeholder(s)` | Dimmed hint text when input is empty |
| `.default(s)` | Value returned on empty input |
| `.required(bool)` | Reject empty input |
| `.validate(fn)` | Custom validation function |
| `.prompt()` | Run the prompt |

**Controls:**
- Type to enter text
- `Backspace` to delete
- `Enter` to submit
- `Esc` to cancel

---

### `select` — Choice Selection

```rust
use viontin_tui::prompts::select;

let color = select("Choose a color:")
    .choices(vec!["Red", "Green", "Blue", "Yellow"])
    .default("Green")
    .required(true)
    .prompt();  // Result<String, String>
```

| Method | Description |
|--------|-------------|
| `.choices(list)` | List of selectable options |
| `.default(s)` | Pre-select a choice by label |
| `.required(bool)` | Require a selection |
| `.prompt()` | Run the prompt |

**Controls:**
- `↑` / `↓` to navigate
- `Enter` to select
- `Esc` to cancel

---

### `confirm` — Yes/No Confirmation

```rust
use viontin_tui::prompts::confirm;

let agreed = confirm("Do you agree?")
    .default(true)     // default to "Yes"
    .required(false)
    .prompt();         // Result<bool, String>
```

| Method | Description |
|--------|-------------|
| `.default(bool)` | Default answer (`true` = Yes, `false` = No) |
| `.required(bool)` | Reject invalid input |
| `.prompt()` | Run the prompt |

**Input:**
- `y` / `yes` → `true`
- `n` / `no` → `false`
- `Enter` → default value
- Any other input → error if required, otherwise default

---

### `password` — Hidden Input

```rust
use viontin_tui::prompts::password;

let pwd = password("Enter password:")
    .placeholder("********")
    .required(true)
    .validate(|s| {
        if s.len() < 8 {
            Err("Minimum 8 characters".into())
        } else {
            Ok(())
        }
    })
    .prompt();  // Result<String, String>
```

| Method | Description |
|--------|-------------|
| `.placeholder(s)` | Dimmed hint when empty |
| `.required(bool)` | Reject empty input |
| `.validate(fn)` | Custom validation |
| `.prompt()` | Run the prompt |

**Controls:**
- Type to enter (masked as `*`)
- `Backspace` to delete
- `Enter` to submit
- `Esc` to cancel

---

### Re-exported `crossterm`

For building custom prompts, `crossterm` is re-exported:

```rust
use viontin_tui::prompts::crossterm;

// Access crossterm's raw terminal, event, and styling APIs
use crossterm::event::{read, Event, KeyCode};
```

---

## Validator

The `validator` module validates command signatures and user input. It is a standalone module (does not depend on the framework's `Validator` trait).

### Signature Validation

```rust
use viontin_tui::validator;

// Validate a signature string
let result = validator::validate_signature("make:controller {name} {--force}");

if result.has_errors() {
    for finding in result.errors() {
        println!("{}", finding);  // [error] C004: Argument name must not be empty
    }
}
```

### Input Validation

```rust
let result = validator::validate_input(
    "greet {name} {--shout}",
    &["World".to_string()],      // provided positional args
    &["shout".to_string()],      // provided flags
    &[],                          // provided options
);

if result.has_errors() {
    // ...
}
```

### Command Name Validation

```rust
let result = validator::validate_command_name("my:command");
assert!(!result.has_errors());
```

### Error Codes

| Code | Description |
|------|-------------|
| C001 | Empty signature |
| C002 | Command name is a token |
| C003 | Unusual characters in command name |
| C004 | Empty argument name |
| C005 | Empty option name |
| C006 | Invalid token format |
| C010 | Missing required arguments |
| C011 | Unknown flag |
| C012 | Unknown option |
| C020 | Empty command name |
| C021 | Command starts with `-` |
| C022 | Command name too long |

### Finding

```rust
use viontin_tui::validator::{Finding, Severity, Outcome};

let finding = Finding::error("C001", "Command signature must not be empty")
    .at("src/commands/mod.rs:12");

println!("{}", finding);
// [error] C001: Command signature must not be empty (at src/commands/mod.rs:12)
```

| Method | Description |
|--------|-------------|
| `Finding::error(code, msg)` | Create an error finding |
| `Finding::warning(code, msg)` | Create a warning finding |
| `Finding::info(code, msg)` | Create an info finding |
| `.at(location)` | Add source location |

### Outcome

```rust
let mut outcome = Outcome::new();
outcome.error("C001", "Something went wrong");
outcome.warning("C010", "Deprecated syntax");

outcome.has_errors();   // true
outcome.errors();       // Vec<&Finding> — only errors
outcome.is_empty();     // false
```

---

## Re-exports from Framework

The TUI crate re-exports the following types from `framework::cli`:

| Type | Description |
|------|-------------|
| `Kernel` | Command registry and dispatcher |
| `Command` | Trait for CLI commands |
| `Input` | Parsed CLI arguments and options |
| `Output` | Styled terminal output (line, info, warn, error, success, table) |
| `ExitCode` | Exit status codes (SUCCESS, FAILURE, INVALID) |

These are the same types used by `viontin-cli`. The TUI crate adds terminal-specific features on top.

---

## Complete Example

```rust
use viontin_tui::prompts::{text, select, confirm};
use viontin_tui::styling::*;
use viontin::prelude::*;

fn main() {
    println!("{}", bold(blue("Project Setup\n")));

    let name = text("Application name:")
        .placeholder("my-app")
        .required(true)
        .prompt()
        .unwrap();

    let template = select("Template:")
        .choices(vec!["web", "cli", "tui", "library"])
        .default("web")
        .prompt()
        .unwrap();

    let git = confirm("Initialize git repository?")
        .default(true)
        .prompt()
        .unwrap();

    let db = select("Database:")
        .choices(vec!["none", "sqlite", "postgres", "mysql"])
        .default("sqlite")
        .prompt()
        .unwrap();

    println!("\n{}", green("✓ Project configured!"));
    println!("  Name:     {}", bold(&name));
    println!("  Template: {}", template);
    println!("  Git:      {}", if git { "Yes" } else { "No" });
    println!("  Database: {}", db);
}
```