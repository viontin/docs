> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# WebSocket

**Module:** `viontin_framework::ws`

Full WebSocket server implementation — handshake, frame encode/decode, and event-driven handler — all hand-written (SHA-1, Base64 included, zero external deps).

---

## Quick Start

```rust
use viontin::prelude::*;
use viontin::ws::{Message, WebSocketHandler};

struct EchoHandler;

impl WebSocketHandler for EchoHandler {
    fn on_message(&self, key: &str, msg: Message) -> Vec<Message> {
        println!("[{}] Received: {:?}", key, msg);
        vec![msg]  // echo back
    }
}

fn main() {
    boot()
        .ws("/echo", EchoHandler)
        .serve("127.0.0.1:3000");
}
```

---

## WebSocketHandler Trait

```rust
pub trait WebSocketHandler: Send + Sync + 'static {
    fn on_open(&self, key: &str) {}
    fn on_message(&self, key: &str, msg: Message) -> Vec<Message> { Vec::new() }
    fn on_close(&self, key: &str, code: Option<u16>, reason: Option<&str>) {}
    fn on_error(&self, key: &str, err: &str) {}
}
```

| Method | Description |
|--------|-------------|
| `on_open(key)` | Connection established |
| `on_message(key, msg)` | Message received — return replies to send back |
| `on_close(key, code, reason)` | Connection closed |
| `on_error(key, err)` | Error occurred |

---

## Message

```rust
pub enum Message {
    Text(String),
    Binary(Vec<u8>),
    Ping(Vec<u8>),
    Pong(Vec<u8>),
    Close(Option<u16>, Option<String>),
}
```

---

## Opcode

```rust
pub enum Opcode {
    Text, Binary, Close, Ping, Pong, Continue,
}
```

---

## WebSocketConfig

```rust
pub struct WebSocketConfig {
    pub max_message_size: usize,   // default: 1 MB
    pub max_frame_size: usize,     // default: 64 KB
}
```

```rust
let config = WebSocketConfig {
    max_message_size: 1024 * 1024,
    max_frame_size: 65536,
};
```

---

## WsRouter

```rust
pub struct WsRouter {
    routes: HandlerMap,
}
```

```rust
let router = ws_router()
    .ws("/chat", ChatHandler)
    .ws_with_config("/video", WebSocketConfig { max_message_size: 10_000_000, ..Default::default() }, VideoHandler);
```

---

## WsServer

Created by attaching a `WsRouter` to a `Router`:

```rust
pub struct WsServer {
    router: Arc<Router>,
    routes: HandlerMap,
}
```

Handles both regular HTTP and WebSocket connections on the same port. Non-WebSocket requests are passed to the HTTP `Router`.

---

## Chat Example

```rust
use std::sync::{Arc, Mutex};
use viontin::prelude::*;
use viontin::ws::{Message, WebSocketHandler};

struct ChatRoom {
    messages: Mutex<Vec<String>>,
}

impl WebSocketHandler for ChatRoom {
    fn on_message(&self, key: &str, msg: Message) -> Vec<Message> {
        match msg {
            Message::Text(text) => {
                let formatted = format!("[{}]: {}", key, text);
                self.messages.lock().unwrap().push(formatted.clone());
                // Broadcast to all — in production, track connections
                vec![Message::Text(formatted)]
            }
            other => vec![other],
        }
    }

    fn on_open(&self, key: &str) {
        println!("Chat: {} joined", key);
    }

    fn on_close(&self, key: &str, _code: Option<u16>, _reason: Option<&str>) {
        println!("Chat: {} left", key);
    }
}

fn main() {
    boot()
        .ws("/chat", ChatRoom { messages: Mutex::new(Vec::new()) })
        .serve("127.0.0.1:3000");
}
```

---

## Implementation Details

The WebSocket implementation is fully self-contained:

| Component | Implementation |
|-----------|---------------|
| SHA-1 | Hand-rolled (RFC 3174) — `ws/sha1` |
| Base64 | Hand-rolled — `ws/base64` |
| Frame encoding | Opcode + mask + length + payload |
| Frame decoding | FIN bit, opcode, mask, extended length |
| Handshake | Sec-WebSocket-Key + magic GUID → SHA-1 → Base64 |

No external dependency on `sha1`, `base64`, or `tokio-tungstenite` crates.
