# Validation
> Last updated: 2026-05-25

**Module:** `viontin_framework::http::form_request`  
**Trait:** `FormRequest`

The validation system provides request authorization and input validation through the `FormRequest` trait. It is the last line of defense before data reaches your controller or service.

---

## Philosophy

- **Decoupled** — validation lives in dedicated types, separate from controllers.
- **Authorization-aware** — `authorize()` runs before `validate()`.
- **Optional** — validate inline in controllers if you prefer. `FormRequest` is pure convenience.
- **Extensible** — override `validate()` for custom logic beyond the built-in rules.

---

## Trait Definition

```rust
pub trait FormRequest: Debug + Send + Sync {
    /// Authorize the request. Return false to deny access (403).
    fn authorize(&self) -> bool { true }

    /// Validation rules as (field, rule) pairs.
    fn rules(&self) -> Vec<(&str, &str)> { Vec::new() }

    /// Custom error messages for rules.
    fn messages(&self) -> Vec<(&str, &str)> { Vec::new() }

    /// Validate the request.
    fn validate(&self, req: &Request) -> Result<(), Vec<String>>;

    /// Validate and return a 422/403 response on failure.
    fn validate_or_reject(&self, req: &Request) -> Result<(), Response>;
}
```

---

## Usage

### Basic Form Request

```rust
use viontin::FormRequest;
use viontin::fw::http::Request;

pub struct CreateUserRequest;

impl FormRequest for CreateUserRequest {
    fn authorize(&self) -> bool {
        true // anyone can register
    }

    fn rules(&self) -> Vec<(&str, &str)> {
        vec![
            ("name", "required|min:2"),
            ("email", "required|email"),
            ("password", "required|min:8"),
        ]
    }
}
```

### In a Controller

```rust
fn store(req: Request) -> Response {
    let form = CreateUserRequest;

    // Returns Err(Response) on failure with proper status code
    form.validate_or_reject(&req)?;

    // Proceed — request is valid and authorized
    // ...
}
```

### Custom Validation Logic

```rust
impl FormRequest for UpdateProfileRequest {
    fn rules(&self) -> Vec<(&str, &str)> {
        vec![
            ("bio", "max:500"),
            ("website", "required"),
        ]
    }

    fn validate(&self, req: &Request) -> Result<(), Vec<String>> {
        // Run built-in rules first
        let builtin = self.default_validate(req); // conceptual

        // Custom logic
        let body = req.body_str();
        let mut errors = Vec::new();

        if body.contains("spam") {
            errors.push("Content not allowed".into());
        }

        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }
}
```

### Entity-Level Validation

Validation also happens at the `Entity` level via `Entity::validate()`, which is called automatically by `Repository::save()`:

```rust
impl Entity for User {
    fn validate(&self) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();
        if self.name.len() < 2 {
            errors.push("Name too short".into());
        }
        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }
}
```

---

## Available Rules

| Rule | Description | Example |
|------|-------------|---------|
| `required` | Field must be present and non-empty | `"name": "required"` |
| `min:N` | Minimum length/ value | `"password": "min:8"` |
| `max:N` | Maximum length/value | `"bio": "max:500"` |
| `email` | Must contain `@` and `.` | `"email": "email"` |
| `numeric` | Must parse as number | `"age": "numeric"` |

Rules can be combined with `|`:

```rust
vec![
    ("username", "required|min:3|max:50"),
    ("age", "required|numeric|min:18"),
]
```

---

## Flow

```
Request → FormRequest::authorize()  → 403 if denied
        → FormRequest::validate()   → 422 if invalid
        → Controller::before()      → custom rejection
        → Controller::action()
        → Service::before()
        → Repository::before_save()
        → Entity::validate()        → last check
        → Database write
```

---

## See Also

- [Entity](../data/entity) — entity-level validation via `Entity::validate()`
- [Controller](../core/controller) — using FormRequest in controllers
- [Web App](../app-types/web-app) — request and response types