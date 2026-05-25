> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Macros

Compile-time content embedding and template engine for HTML, Markdown, JSON, JavaScript, and TypeScript.

---

## Quick Reference

| Macro | Input | Output | Use Case |
|-------|-------|--------|----------|
| `html!(...)` | Inline template with `{{var}}` | `String` via `.render()` | Dynamic HTML with variables |
| `md!(...)` | Inline Markdown with `{{var}}` | `String` via `.render()` | Dynamic Markdown rendered to HTML |
| `json!({...})` | Inline JSON expression | `serde_json::Value` | JSON literals at compile time |
| `include_html!("path")` | File path (`.html`) | `&'static str` | Static HTML file embedding |
| `include_md!("path")` | File path (`.md`) | `String` | Markdown file rendered to HTML |
| `include_json!("path")` | File path (`.json`) | `serde_json::Value` | JSON file embedding + parsing |
| `include_js!("path")` | File path (`.js`) | `&'static str` | JavaScript file embedding |
| `include_ts!("path")` | File path (`.ts`) | `&'static str` | TypeScript file embedding |

---

## html!() — Inline Templates with Interpolation

Build HTML strings with variable substitution and conditionals:

```rust
use viontin::html;

// Variable substitution
let out = html!("<h1>{{title}}</h1>")
    .with("title", "Dashboard")
    .render();
// → <h1>Dashboard</h1>

// Conditionals
let out = html!("<div class=\"{{active ? \"on\" : \"off\"}}\">")
    .with("active", true)
    .render();
// → <div class="on">

// Multiple variables
let out = html!("<h1>{{title}}</h1><p>Welcome, {{name}}!</p>")
    .with("title", "Profile")
    .with("name", "Alice")
    .render();
// → <h1>Profile</h1><p>Welcome, Alice!</p>
```

### Chain Methods

| Method | Description |
|--------|-------------|
| `.with(key, value)` | Bind a variable (returns self for chaining) |
| `.render()` | Process template and return `String` |

### Variable Syntax

| Pattern | Description |
|---------|-------------|
| `{{key}}` | Replaced with `.with("key", value)` — value is converted via `ToString` |
| `{{key ? "a" : "b"}}` | Conditional — if `key` is `true`, renders `a`; otherwise `b` |

---

## md!() — Inline Markdown with Interpolation

Write Markdown inline, render to HTML with the same template engine:

```rust
use viontin::md;

let out = md!("# {{title}}")
    .with("title", "Getting Started")
    .render();
// → <h1>Getting Started</h1>

// Full example
let doc = md!("# {{page}}

{{show ? "Active" : "Inactive"}}

- {{item1}}
- {{item2}}")
    .with("page", "Features")
    .with("show", true)
    .with("item1", "Lightning fast")
    .with("item2", "Zero dependencies")
    .render();
```

### Markdown Syntax Supported

| Element | Syntax | Output |
|---------|--------|--------|
| Heading 1 | `# Text` | `<h1>Text</h1>` |
| Heading 2 | `## Text` | `<h2>Text</h2>` |
| Heading 3 | `### Text` | `<h3>Text</h3>` |
| Bold | `**text**` or `__text__` | `<strong>text</strong>` |
| Italic | `*text*` | `<em>text</em>` |
| List | `- item` or `* item` | `<ul><li>item</li></ul>` |
| Blockquote | `> text` | `<blockquote><p>text</p></blockquote>` |
| Code block | ` ``` ` ... ` ``` ` | `<pre><code>...</code></pre>` |
| Paragraph | Plain text | `<p>text</p>` |

---

## json!() — Inline JSON Values

```rust
use viontin::json;

let val = json!({"name": "Alice", "age": 30, "tags": ["admin", "user"]});
// → serde_json::Value

// Access fields
let name = val["name"].as_str().unwrap();
```

> `json!()` delegates to `serde_json::json!()` — the same compile-time validated JSON literal syntax.

---

## include_* — File Embedding

All `include_*` macros embed file contents at **compile time** with zero file I/O at runtime:

```rust
use viontin::{include_html, include_md, include_js, include_json};

// Include raw HTML
let page: &str = include_html!("templates/index.html");

// Include and render Markdown
let doc: String = include_md!("docs/guide.md");
// Converts to HTML at runtime

// Include and parse JSON
let config: serde_json::Value = include_json!("config/app.json");
// Validated at compile time via include_str! — file must be valid JSON

// Include JavaScript
let script: &str = include_js!("assets/app.js");

// Include TypeScript
let component: &str = include_ts!("components/app.ts");
```

### Path Resolution

Paths are resolved relative to the source file where the macro is called — same behavior as `include_str!()`.

---

## Complete Example

```rust
use viontin::{html, include_html};

// Static template from file
let layout = include_html!("layouts/base.html");

// Dynamic section with variables
let content = html!("<main><h1>{{title}}</h1><p>{{body}}</p></main>")
    .with("title", "Hello")
    .with("body", "World")
    .render();

// Combine static + dynamic
let page = layout.replace("{{content}}", &content);
```

---

## See Also

- [Web App](platforms/webapp) — Using templates in web responses
- [Architecture](architecture/architecture) — Content embedding in the platform
- [Getting Started](getting-started) — Quick start with templates
