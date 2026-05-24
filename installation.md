# Installation

> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

---

## Prerequisites

- **Rust** — Viontin targets Rust edition 2024. Install via [rustup](https://rustup.rs).

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

---

## Method 1: Install the CLI (Recommended)

```bash
cargo install viontin
```

This installs the `viontin` binary — 42 commands, zero `cargo` dependency at runtime. After installation:

```bash
viontin --help
```

### From Local Build

```bash
git clone <repo-url> viontin
cd viontin
bash scripts/install.sh
```

Options:
- `--path ./cli` — install from a custom path
- `--release` — install in release mode

---

## Method 2: Add as a Dependency

Add the meta-crate to your `Cargo.toml`:

```toml
[dependencies]
viontin = { path = "path/to/repos/framework/crates/viontin" }
```

Or reference individual crates:

```toml
[dependencies]
viontin-framework = { path = "path/to/repos/framework/crates/framework" }
viontin-tui = { path = "path/to/repos/framework/crates/tui", features = ["prompts"] }
```

---

## Method 3: Start from an Example

```bash
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
