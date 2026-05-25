> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# CSRF Protection

**Module:** `viontin_framework::csrf`

Cross-Site Request Forgery (CSRF) protection — generates tokens, stores them in sessions, and validates incoming requests for state-changing methods.

---

## Quick Start

```rust
use viontin::prelude::*;

let csrf = CsrfManager::new(CsrfConfig::default());
let mut session = Session::memory();

// Get or generate token
let token = csrf.get_or_create(&mut session);

// Include in forms:
// <input type="hidden" name="_csrf_token" value="{{ token }}">

// Validate on POST/PUT/PATCH/DELETE
if csrf.validate(&session, &incoming_token) {
    // proceed
}
```

---

## CsrfManager

```rust
pub struct CsrfManager {
    config: CsrfConfig,
}
```

### Methods

| Method | Description |
|--------|-------------|
| `generate()` | Create a new random token |
| `get_or_create(session)` | Return existing token from session, or generate one |
| `validate(session, token)` | Constant-time comparison against session token |
| `needs_protection(method, path)` | Check if request needs CSRF protection |
| `config()` | Access current configuration |

### Constant-Time Comparison

Validation uses XOR-based constant-time comparison to prevent timing attacks:

```rust
let mut result = 0u8;
for (a, b) in stored.bytes().zip(token.bytes()) {
    result |= a ^ b;
}
result == 0
```

---

## CsrfConfig

```rust
pub struct CsrfConfig {
    pub token_length: usize,        // default: 32
    pub header_name: String,        // default: "X-CSRF-Token"
    pub form_key: String,           // default: "_csrf_token"
    pub protected_methods: Vec<&'static str>, // POST, PUT, PATCH, DELETE
    pub excluded_paths: Vec<String>, // e.g. ["/webhook", "/api/public"]
}
```

### Default

```rust
CsrfConfig {
    token_length: 32,
    header_name: "X-CSRF-Token".into(),
    form_key: "_csrf_token".into(),
    protected_methods: vec!["POST", "PUT", "PATCH", "DELETE"],
    excluded_paths: vec![],
}
```

### Custom Configuration

```rust
let config = CsrfConfig {
    token_length: 64,
    header_name: "X-CSRF-TOKEN".into(),
    excluded_paths: vec!["/webhook".into(), "/api/".into()],
    ..Default::default()
};

let csrf = CsrfManager::new(config);
```

---

## Integration with HTTP

```rust
fn show_form(mut session: Session) -> Response {
    let csrf = CsrfManager::default();
    let token = csrf.get_or_create(&mut session);
    session.save();

    Response::html(format!(
        r#"<form method="POST">
            <input type="hidden" name="_csrf_token" value="{}" />
            <button>Submit</button>
          </form>"#,
        token
    ))
}

fn handle_post(request: Request, mut session: Session) -> Response {
    let csrf = CsrfManager::default();

    // Check CSRF
    let token = request.header("X-CSRF-Token")
        .or_else(|| request.body_str()) // or from form body
        .unwrap_or("");

    if !csrf.validate(&session, token) {
        return Response::html("Invalid CSRF token")
            .status(StatusCode::BAD_REQUEST);
    }

    Response::html("Success")
}
```

---

## Protecting Routes

```rust
fn check_csrf(request: &Request, session: &Session) -> bool {
    let csrf = CsrfManager::default();

    if !csrf.needs_protection(&request.method.as_str(), &request.uri.path) {
        return true; // GET/HEAD/OPTIONS don't need protection
    }

    let token = request.header("X-CSRF-Token")
        .unwrap_or("");

    csrf.validate(session, token)
}
```