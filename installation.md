# Installation
> Last updated: 2026-05-25

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

---

## Prerequisites

- **Rust** — Viontin targets Rust edition 2024. Install via [rustup](https://rustup.rs).

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

---

## Method 1: Install the CLI (Recommended)

### From Source (Git)

```bash
git clone https://github.com/viontin/viontin.git viontin
cd viontin
bash scripts/install.sh --release
```

After installation:

```bash
viontin --help
```

### Install Script Options

| Flag | Description |
|------|-------------|
| `--path ./cli` | Install from a custom path |
| `--release` | Install in release mode |

---

## Method 2: Add as a Dependency

Viontin is distributed via **GitHub**, not crates.io. This allows flexible licensing for the future.

### From Git (recommended for users)

```toml
[dependencies]
viontin = { git = "https://github.com/viontin/viontin" }
```

With a specific version tag:

```toml
[dependencies]
viontin = { git = "https://github.com/viontin/viontin", tag = "v0.1.0" }
```

### From Local Path (development within the monorepo)

```toml
[dependencies]
viontin = { path = "products/viontin/crates/viontin" }
```

Or individual crates:

```toml
[dependencies]
viontin-framework = { git = "https://github.com/viontin/framework" }
viontin-tui = { git = "https://github.com/viontin/tui", features = ["prompts"] }
viontin-orm = { git = "https://github.com/viontin/orm" }
```

---

## Method 3: Start from an Example

```bash
# Clone the examples repo
git clone https://github.com/viontin/examples.git

# Minimal starter
cp -r examples/viontin-zero my-app

# Feature demo (with TailwindCSS + serde)
cp -r examples/viontin-alpha my-app
```

---

## Verify Installation

```bash
viontin new test-project
cd test-project
viontin dev
```

If the dev server starts without errors, installation is complete.
