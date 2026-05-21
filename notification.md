# Notifications

**Module:** `viontin_framework::notif`

Multi-channel notification system — send notifications via mail, database, or custom channels.

---

## Architecture

```
Notif (facade)
  │
  ├── MailChannel     → sends via mail system
  ├── DatabaseChannel → stores in memory (for testing)
  └── Custom channels → implement Channel trait
```

---

## Notifiable Trait

```rust
pub trait Notifiable: Debug + Send + Sync {
    fn route_notification(&self, channel: &str) -> Option<String>;
}
```

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct User {
    id: i64,
    email: String,
}

impl Notifiable for User {
    fn route_notification(&self, channel: &str) -> Option<String> {
        match channel {
            "mail" => Some(self.email.clone()),
            "database" => Some(self.id.to_string()),
            _ => None,
        }
    }
}
```

---

## Notification Trait

```rust
pub trait Notification: Debug + Send + Sync {
    fn channels(&self) -> Vec<&'static str>;
    fn to_mail(&self, notifiable: &dyn Notifiable) -> Option<String> { None }
    fn to_database(&self, notifiable: &dyn Notifiable) -> Option<String> { None }
    fn to_slack(&self, notifiable: &dyn Notifiable) -> Option<String> { None }
}
```

```rust
#[derive(Debug)]
struct WelcomeNotification;

impl Notification for WelcomeNotification {
    fn channels(&self) -> Vec<&'static str> {
        vec!["mail", "database"]
    }

    fn to_mail(&self, _notifiable: &dyn Notifiable) -> Option<String> {
        Some("<h1>Welcome!</h1>".into())
    }

    fn to_database(&self, notifiable: &dyn Notifiable) -> Option<String> {
        Some(format!("Welcome user {}", notifiable.route_notification("database").unwrap_or_default()))
    }
}
```

---

## Channel Trait

```rust
pub trait Channel: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn send(&self, notifiable: &dyn Notifiable, notification: &dyn Notification) -> Result<(), String>;
}
```

---

## Built-in Channels

### MailChannel

Sends via the mail system:

```rust
let mailer = Mail::new(LogTransport);
let channel = MailChannel::new(mailer);
```

### DatabaseChannel

Stores in memory (for testing):

```rust
let channel = DatabaseChannel::new();
channel.all();  // Vec<(String, String)> — (route, data)
```

---

## Notif Facade

```rust
use viontin::prelude::*;

let mut notif = Notif::new();
notif.add_channel("mail", Box::new(MailChannel::new(Mail::new(LogTransport))));
notif.add_channel("database", Box::new(DatabaseChannel::new()));

let user = User { id: 1, email: "alice@example.com".into() };
notif.send(&user, &WelcomeNotification).unwrap();
```

---

## Complete Example

```rust
use viontin::prelude::*;

#[derive(Debug)]
struct OrderConfirmation {
    order_id: String,
}

impl Notification for OrderConfirmation {
    fn channels(&self) -> Vec<&'static str> {
        vec!["mail"]
    }

    fn to_mail(&self, _notifiable: &dyn Notifiable) -> Option<String> {
        Some(format!("<h1>Order {} confirmed!</h1>", self.order_id))
    }
}

fn main() {
    let mut notif = Notif::new();
    notif.add_channel("mail", Box::new(MailChannel::new(Mail::new(LogTransport))));

    let user = User { id: 42, email: "user@example.com".into() };
    notif.send(&user, &OrderConfirmation { order_id: "ORD-456".into() }).unwrap();
}
```
