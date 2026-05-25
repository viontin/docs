# Getting Started
> Last updated: 2026-05-25

> **Experimental Project** — This is an experimental project under active development. APIs are unstable, documentation is incomplete, and breaking changes may occur without notice. Not recommended for production use.

---

This guide walks you through creating your first Viontin application — a web server, a CLI command, and a terminal prompt — in under five minutes.

---

## 1. Create a Project

```bash
viontin new hello-viontin
cd hello-viontin
```

This scaffolds a complete project with:

```
hello-viontin/
├── Cargo.toml
├── .env
├── config/
│   └── config.json
├── html/
├── src/
│   ├── main.rs
│   └── lib.rs
└── routes/
    └── mod.rs
```

---

## 2. Your First Web Server

Open `src/main.rs`:

```rust
use viontin::prelude::*;

fn main() {
    boot()
        .get("/", |_req| Response::html("<h1>Hello, Viontin!</h1>"))
        .get("/hello/:name", |req| {
            let name = req.param("name").unwrap_or("world");
            Response::html(format!("<h1>Hello, {}!</h1>", name))
        })
        .serve("127.0.0.1:3000");
}
```

Run it:

```bash
viontin dev
```

Visit `http://127.0.0.1:3000/hello/viontin`.

---

## 3. Your First CLI Command

Add a command in `src/main.rs`:

```rust
use viontin::prelude::*;

struct GreetCommand;

impl Command for GreetCommand {
    fn signature(&self) -> &Signature {
        &Signature::new("greet {name} {--shout}")
    }

    fn description(&self) -> &str {
        "Greet someone"
    }

    fn handle(&self, input: Input, output: Output) -> ExitCode {
        let name = input.argument("name").unwrap_or("world");
        let mut msg = format!("Hello, {}!", name);
        if input.has_flag("shout") {
            msg = msg.to_uppercase();
        }
        output.line(&msg);
        ExitCode::SUCCESS
    }
}

fn main() {
    boot()
        .command(GreetCommand)
        .run_with(|_ctx| {});
}
```

Run it:

```bash
cargo run -- greet Viontin --shout
# Output: HELLO, VIONTIN!
```

---

## 4. Your First TUI Prompt

Using the TUI toolkit:

```rust
use viontin::prelude::*;
use viontin_tui::prompts::{text, confirm};

fn main() {
    let name = text("What is your name?").prompt().unwrap_or("world".into());
    let happy = confirm("Are you happy today?").prompt().unwrap_or(false);

    let mood = if happy { "😊" } else { "🤔" };
    println!("Hello, {}! {}", name, mood);
}
```

Run it:

```bash
cargo run
```

---

## 5. What's Next

| Guide | Description |
|-------|-------------|
| [Installation](installation) | Full installation options |
| [Web App](platforms/webapp) | Routes, middleware, responses, templates |
| [Native App](platforms/desktop) | CLI commands, signatures, input/output |
| [Terminal App](platforms/terminal) | TUI prompts, styling, interactive workflows |

## API Reference

### Boot Builder

| Method | Description |
|--------|-------------|
| `boot()` | Create a new Boot builder |
| `.provider(P)` | Register a service provider |
| `.command(C)` | Register a CLI command |
| `.gem(G)` | Register a gem plugin (implements `GemBuilder` + `GemBinding`) |
| `.middleware(M)` | Register global middleware |
| `.get(path, handler)` | Register HTTP GET route |
| `.post(path, handler)` | Register HTTP POST route |
| `.any(path, handler)` | Register route for any HTTP method |
| `.ws(path, handler)` | Register WebSocket handler |
| `.serve(addr)` | Start HTTP server (shortcut) |
| `.entry(f)` | Define application entry point |
| `.run()` | Finalize + execute |
| `.run_with(f)` | Shortcut for `entry(f).run()` |