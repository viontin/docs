> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# Support Utilities

**Module:** `viontin_framework::support`

String manipulation, URL encoding/decoding, random generation, and utility traits.

---

## String Utilities

### truncate

```rust
use viontin::prelude::*;

truncate("hello world", 5);  // "hello..."
truncate("short", 20);      // "short" (unchanged)
```

### Case Conversion

```rust
kebab_case("HelloWorld");     // "hello-world"
kebab_case("user_name");      // "user-name"
snake_case("HelloWorld");     // "hello_world"
snake_case("kebab-case");     // "kebab_case"
pascal_case("hello_world");   // "HelloWorld"
pascal_case("kebab-case");    // "KebabCase"
camel_case("hello_world");    // "helloWorld"
```

### slug

```rust
slug("Hello World!");         // "hello-world"
slug("Rust is great!!");      // "rust-is-great"
slug("  spaces  ");           // "spaces"
```

### pluralize

```rust
pluralize("user");    // "users"
pluralize("box");     // "boxes"
pluralize("cherry");  // "cherries"
pluralize("beach");   // "beaches"
pluralize("boy");     // "boys"
```

### random

```rust
random(16);  // "a3x9k2m7bn4q1w8e" (alphanumeric, a-z + 0-9)
```

---

## URL Utilities

```rust
use viontin::prelude::*;

// Encoding
url_encode("hello world");           // "hello%20world"
url_encode("a=b&c=d");               // "a%3Db%26c%3Dd"

// Decoding
url_decode("hello%20world");         // "hello world"
url_decode("a%3Db");                 // "a=b"

// Query strings
parse_query("page=1&sort=asc");      // [("page", "1"), ("sort", "asc")]
build_query(&[("q", "rust"), ("page", "2")]);  // "q=rust&page=2"

// URL validation
is_valid_url("https://example.com"); // true
is_valid_url("ftp://example.com");   // false
```

---

## Hasher Trait

```rust
pub trait Hasher: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn hash(&self, value: &str) -> String;
    fn verify(&self, value: &str, hash: &str) -> bool;
    fn needs_rehash(&self, _hash: &str) -> bool { false }
}
```

See [Hashing](hashing) for details.

---

## Encrypter Trait

```rust
pub trait Encrypter: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn encrypt(&self, plaintext: &str) -> String;
    fn decrypt(&self, ciphertext: &str) -> Result<String, String>;
    fn keygen() -> String;
}
```

See [Encryption](encryption) for details.

---

## Hash Utilities

```rust
use viontin::prelude::*;

hex_digest("hello");          // hex hash string
quick_hash(&42);              // u64 hash
random_token(32);             // random alphanumeric token
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn create_slug(title: &str) -> String {
    slug(&truncate(title, 80))
}

fn build_api_url(base: &str, path: &str, params: &[(&str, &str)]) -> String {
    let encoded_path = path.split('/')
        .map(|seg| url_encode(seg))
        .collect::<Vec<_>>()
        .join("/");

    if params.is_empty() {
        format!("{}{}", base, encoded_path)
    } else {
        format!("{}{}?{}", base, encoded_path, build_query(params))
    }
}

let slug = create_slug("Hello World! This is a test");
assert_eq!(slug, "hello-world-this-is-a-test");

let url = build_api_url("https://api.example.com", "/users/search", &[("q", "rust lang")]);
assert_eq!(url, "https://api.example.com/users/search?q=rust%20lang");
```