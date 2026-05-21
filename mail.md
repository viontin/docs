> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Mail

**Module:** `viontin_framework::mail`

Mail sending abstraction with swappable transports.

---

## Architecture

```
Mail (facade)
  │
  └── Box<dyn Mailer>
        │
        ├── LogTransport     — log to stdout (development)
        └── ArrayTransport   — in-memory (testing)
```

---

## Envelope

```rust
pub struct Envelope {
    pub from: Option<String>,
    pub to: Vec<String>,
    pub cc: Vec<String>,
    pub bcc: Vec<String>,
    pub subject: String,
    pub html_body: Option<String>,
    pub text_body: Option<String>,
    pub attachments: Vec<Attachment>,
}
```

```rust
let envelope = Envelope {
    from: Some("noreply@example.com".into()),
    to: vec!["user@example.com".into()],
    subject: "Welcome".into(),
    html_body: Some("<h1>Welcome!</h1>".into()),
    text_body: Some("Welcome!".into()),
    ..Default::default()
};
```

---

## Attachment

```rust
pub struct Attachment {
    pub filename: String,
    pub content: Vec<u8>,
    pub mime_type: String,
}
```

```rust
let attachment = Attachment::new(
    "report.pdf",
    pdf_bytes,
    "application/pdf",
);
```

---

## Mailer Trait

```rust
pub trait Mailer: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn send(&self, envelope: &Envelope) -> Result<(), String>;
    fn is_usable(&self) -> bool { true }
}
```

---

## Transports

### LogTransport

Prints email content to stdout:

```rust
let mail = Mail::new(LogTransport);
mail.send(&envelope).unwrap();
// [mail] To: user@example.com
// [mail] Subject: Welcome
// [mail] HTML: 19 bytes
// [mail] Text: Welcome!
```

Best for development.

### ArrayTransport

Collects sent emails in memory:

```rust
let transport = ArrayTransport::new();
let mail = Mail::new(transport);

mail.send(&envelope).unwrap();

let sent = transport.sent_emails();
assert_eq!(sent.len(), 1);
transport.clear();
```

Best for testing.

---

## Mail Facade

```rust
use viontin::prelude::*;

let mail = Mail::new(LogTransport);
mail.send(&envelope);
mail.mailer().name();  // "log"
```

---

## Complete Example

```rust
use viontin::prelude::*;

fn send_welcome(email: &str) {
    let envelope = Envelope {
        from: Some("welcome@example.com".into()),
        to: vec![email.into()],
        cc: vec![],
        bcc: vec![],
        subject: "Welcome to Our Service!".into(),
        html_body: Some("<h1>Welcome!</h1><p>Thanks for joining.</p>".into()),
        text_body: Some("Welcome! Thanks for joining.".into()),
        attachments: vec![
            Attachment::new("terms.pdf", vec![], "application/pdf"),
        ],
    };

    let mail = Mail::new(LogTransport);
    mail.send(&envelope).unwrap();
}
```
