# Viontin CLI Reference

**33 commands** — zero `cargo` dependency at runtime.

---

## Overview

```
viontin <command> [options] [arguments]
```

The CLI is organized into five levels of maturity:

| Level | Category | Commands |
|-------|----------|---------|
| 0 | Core | `new`, `init`, `build`, `dev`, `run`, `check`, `test`, `add` |
| — | Cargo Management | `clean`, `doc`, `fix`, `bench`, `tree`, `package`, `metadata` |
| — | Publishing | `publish`, `update`, `install`, `uninstall`, `search` |
| — | Code Quality | `fmt`, `clippy` |
| 1 | Scaffolding | `make:controller`, `make:middleware`, `make:model`, `make:route`, `make:command`, `make:event`, `make:job`, `make:mail`, `make:notification`, `make:query`, `make:module` |
| 2 | Domains | `make:domain`, `inspect` (with `--domains`) |

---

## Level 0 — Core Commands

### `new`

Scaffold a new Rust project.

```bash
viontin new <name> [--with <dirs>] [--force]
```

| Argument | Description |
|----------|-------------|
| `name` | Project name (required) |
| `--with` | Comma-separated list of subdirectories to create in `src/` |
| `--force` | Overwrite existing directory |

```bash
viontin new my-app
viontin new my-app --with controllers,models,routes
viontin new my-app --force
```

Runs `cargo init` internally, then creates any additional directories specified by `--with`.

---

### `init`

Initialize a Rust project in the current directory.

```bash
viontin init [--lib] [--name <name>]
```

| Option | Description |
|--------|-------------|
| `--lib` | Initialize a library crate instead of binary |
| `--name` | Package name (defaults to directory name) |

---

### `build`

Build the project with cargo.

```bash
viontin build [--release]
```

| Option | Description |
|--------|-------------|
| `--release` | Build in release mode |

---

### `dev`

Run the project in development mode with file watching.

```bash
viontin dev [--port <port>]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--port` | `3000` | Port for the development server |

Automatically watches `src/` for `.rs` file changes and restarts the process. Skips the `target/` directory.

---

### `run`

Run a project command via cargo.

```bash
viontin run <command> [args...]
```

| Argument | Description |
|----------|-------------|
| `command` | Command name to run inside the project |
| `args` | Additional arguments passed through |

```bash
viontin run my-command --flag value
# Equivalent to: cargo run -- my-command --flag value
```

---

### `check`

Type-check the project or verify architecture boundaries.

```bash
viontin check [--arch]
```

| Option | Description |
|--------|-------------|
| `--arch` | Scan domain boundaries for import violations |

Without `--arch`: runs `cargo check`.

With `--arch`: scans `src/domain/*/` for cross-domain import violations:
- **Error** (✘): Import from a domain NOT in the `allows` list
- **Warning** (⚠): Import from a domain IN the `allows` list (visible for review)

Output:

```
Architecture Check

Found 3 domain(s):
  ✓ billing
  ✓ order
  ✗ payment

✘ billing → notification (NOT allowed)
    in: src/domain/billing/routes/invoice.rs
    import: notification::send_invoice_notification
```

---

### `test`

Run tests with cargo.

```bash
viontin test
```

---

### `add`

Add a Rust dependency to `Cargo.toml`.

```bash
viontin add <package> [--path <path>] [--git <url>]
```

| Argument | Description |
|----------|-------------|
| `package` | Package name, optionally with `@version` suffix |
| `--path` | Local path dependency |
| `--git` | Git repository URL |

```bash
viontin add serde
viontin add serde@1.0
viontin add serde --path ./libs/serde
viontin add serde --git https://github.com/serde-rs/serde
viontin add all             # Add all viontin packages
```

Known viontin packages are resolved automatically when inside the monorepo:

| Package | Path |
|---------|------|
| `viontin` | `crates/viontin` |
| `viontin-framework` | `crates/framework` |
| `viontin-tui` | `crates/tui` (with `prompts` feature) |
| `viontin-orm` | `../orm/crates/viontin-orm` |
| `viontin-gems` | `../gems/crates/viontin-gems` |

Supports `viontin add all` to add every viontin package at once.

---

## Cargo Management

These commands wrap `cargo` subcommands:

| Command | Signature | Delegates to |
|---------|-----------|-------------|
| `clean` | `clean` | `cargo clean` |
| `doc` | `doc {--open}` | `cargo doc --open` |
| `fix` | `fix` | `cargo fix` |
| `bench` | `bench` | `cargo bench` |
| `tree` | `tree` | `cargo tree` |
| `package` | `package` | `cargo package` |
| `metadata` | `metadata` | `cargo metadata` |

All require a `Cargo.toml` in the current directory.

---

## Publishing & Registry

| Command | Signature | Delegates to |
|---------|-----------|-------------|
| `publish` | `publish` | `cargo publish` |
| `update` | `update` | `cargo update` |
| `install` | `install {crate}` | `cargo install <crate>` |
| `uninstall` | `uninstall {crate}` | `cargo uninstall <crate>` |
| `search` | `search {query}` | `cargo search <query>` |

`install`, `uninstall`, and `search` do not require a project directory.

---

## Code Quality

| Command | Signature | Delegates to |
|---------|-----------|-------------|
| `fmt` | `fmt` | `cargo fmt` |
| `clippy` | `clippy` | `cargo clippy` |

---

## Level 1 — Scaffolding

All scaffold commands share the same pattern:

```bash
viontin make:<type> <name> [--force]
```

| Argument | Description |
|----------|-------------|
| `name` | Name of the scaffold (automatically converted to PascalCase and snake_case) |
| `--force` | Overwrite existing files |

### `make:controller`

```
make:controller {name} {--force}
```

Scaffolds `src/controllers/<snake>.rs` with a public `index` function.

---

### `make:middleware`

```
make:middleware {name} {--force}
```

Scaffolds `src/middleware/<snake>.rs` with a `Middleware` struct and `handle` method.

---

### `make:model`

```
make:model {name} {--force}
```

Scaffolds `src/models/<snake>.rs` with a `Model` struct deriving `Serialize`/`Deserialize`.

---

### `make:route`

```
make:route {name} {--force}
```

Scaffolds `src/routes/<snake>.rs` with a route handler function returning `Response`.

---

### `make:command`

```
make:command {name} {--force}
```

Scaffolds `src/commands/<snake>.rs` with a CLI `Command` implementation.

The command signature is automatically derived from the name in kebab-case.

---

### `make:event`

```
make:event {name} {--force}
```

Scaffolds `src/events/<snake>.rs` with an event struct.

---

### `make:job`

```
make:job {name} {--force}
```

Scaffolds `src/jobs/<snake>.rs` with a job struct and `handle` method.

---

### `make:mail`

```
make:mail {name} {--force}
```

Scaffolds `src/mail/<snake>.rs` with an `Envelope` builder function.

---

### `make:notification`

```
make:notification {name} {--force}
```

Scaffolds `src/notifications/<snake>.rs` with a `Notification` implementation.

---

### `make:query`

```
make:query {name} {--force}
```

Scaffolds `src/queries/<snake>.rs` with an `execute` function.

---

### `make:module`

```
make:module {name} {--force}
```

Scaffolds `src/<snake>/mod.rs` with a module declaration.

---

## Level 2 — Domains

### `make:domain`

```
make:domain {name} {--force}
```

Scaffolds a bounded context with boundary definition:

```
src/domain/<snake>/
├── mod.rs         # module declarations
├── domain.rs      # Domain definition with allows/provides
└── port.rs        # Public API (ports)
```

Generated `domain.rs`:

```rust
use viontin::Domain;

pub const DEFINITION: Domain = Domain::new("<snake>")
    .allows(&[]);
```

Also updates `src/domain/mod.rs` with the new module declaration.

---

### `inspect`

Show project structure and exports.

```bash
viontin inspect [--models] [--routes] [--commands] [--events]
                [--jobs] [--mail] [--notifications] [--queries]
                [--domains]
```

Without flags: shows the full project structure with all modules and their exports.

With flags: filters to show only the specified module types.

```
Structure
 my-app
 └── src
     ├── controllers/ (1 file)
     │   └── user.rs (UserController, list, create)
     ├── models/ (1 file)
     │   └── user.rs (User)
     └── routes/ (1 file)
         └── mod.rs
```

#### `inspect --domains`

Shows detailed domain structure:

```
Domains
  billing
    domain.rs  ✓   port.rs  ✓
    allows: order, payment
    files: routes/

  order
    domain.rs  ✓   port.rs  ✓
    allows: payment
    files: models/, routes/
```

Detects:
- Whether `domain.rs` and `port.rs` exist
- Parsed `allows` list from `domain.rs`
- Other `.rs` files and subdirectories in the domain

---

## Exit Codes

| Code | Constant | Meaning |
|------|----------|---------|
| `0` | `ExitCode::SUCCESS` | Command completed successfully |
| `1` | `ExitCode::FAILURE` | General failure |
| `2` | `ExitCode::INVALID` | Invalid arguments |

---

## Project Scanner

The `inspect` command uses an automatic scanner that reads `src/` directories and identifies:

- **Modules** — subdirectories under `src/`
- **Files** — `.rs` files within each module
- **Exports** — `pub struct`, `pub fn`, `pub enum`, `pub trait`, and `impl Command for` declarations

Directories containing a `.viontin-ignore` marker file are excluded from scanning.

---

## Global Options

```bash
viontin --help       # Show help
viontin --version    # Show version
```

Available on all commands.
