# Localization (i18n)

**Module:** `viontin_framework::lang`

JSON-based translation system with locale fallback and pluralization.

---

## Quick Start

```rust
use viontin::prelude::*;

// Initialize
let translator = JsonTranslator::new("en")
    .fallback("en");
lang_init(translator);

// Translate
let msg = trans("messages.welcome", &[("name", "Alice")]);
// Returns "Welcome Alice" or the key if not found

// With pluralization
let count = choice("messages.apples", 5, &[]);
```

---

## File Structure

```
lang/
├── en/
│   └── messages.json   → { "welcome": "Welcome :name", "apples": "{0} None|[1,*] :count apples" }
├── id/
│   └── messages.json   → { "welcome": "Selamat datang :name", "apples": "{0} Tidak ada|[1,*] :count apel" }
└── jp/
    └── messages.json   → { "welcome": ":nameさん、ようこそ" }
```

Keys use dot notation: `"file.key"` → `messages.welcome`.

---

## JsonTranslator

```rust
let mut t = JsonTranslator::new("id")  // current locale
    .fallback("en");                     // fallback if key not found

t.load_dir(Path::new("lang/"))?;
```

### Methods

```rust
t.trans("messages.welcome", &[("name", "Alice")]);
t.choice("messages.apples", 5, &[]);
t.locale();         // "id"
t.set_locale("jp");
```

---

## trans()

```rust
let msg = trans("messages.welcome", &[("name", "Alice")]);
```

Parameters use `:name` syntax in translation strings:

```json
{ "welcome": "Hello :name!" }
```

Output: `"Hello Alice!"`

---

## choice() — Pluralization

```rust
choice("messages.apples", 5, &[]);
```

Translation format:

```
{0} None|{1} One apple|[2,*] :count apples
```

| Rule | Format | Example |
|------|--------|---------|
| Exact | `{N}` | `{0}`, `{1}` |
| Range | `[min,*]` | `[2,*]` (2 or more) |
| Range | `[min,max]` | `[2,10]` |
| Wildcard | `*` | catch-all |

```json
{
    "apples": "{0} No apples|{1} One apple|[2,*] :count apples"
}
```

```rust
choice("apples", 0, &[]); // "No apples"
choice("apples", 1, &[]); // "One apple"
choice("apples", 5, &[]); // "5 apples"
```

---

## Global Functions

```rust
// Init once at startup
let t = JsonTranslator::new("en");
lang_init(t);

// Use anywhere
trans("messages.welcome", &[("name", "Alice")]);
choice("messages.apples", 1, &[]);
locale();  // "en"
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn main() {
    let mut t = JsonTranslator::new("id")
        .fallback("en");
    t.load_dir(Path::new("lang/")).ok();
    lang_init(t);

    let msg = trans("messages.welcome", &[("name", "Budi")]);
    println!("{}", msg);

    let count = choice("messages.apples", 3, &[]);
    println!("{}", count);
}
```
