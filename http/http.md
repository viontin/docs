> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.
> Last updated: 2026-05-25

# HTTP Primitives

**Module:** `viontin_framework::http`

Core HTTP types — `Request`, `Response`, `StatusCode`, `Method`, `Headers`, `Uri`, and `Cookie`.

---

## StatusCode

```rust
pub struct StatusCode(pub u16);
```

### Constants

```rust
StatusCode::OK              // 200
StatusCode::CREATED         // 201
StatusCode::NO_CONTENT      // 204
StatusCode::MOVED           // 301 (Moved Permanently)
StatusCode::FOUND           // 302
StatusCode::BAD_REQUEST     // 400
StatusCode::UNAUTHORIZED    // 401
StatusCode::FORBIDDEN       // 403
StatusCode::NOT_FOUND       // 404
StatusCode::SERVER_ERROR    // 500
```

### Methods

```rust
let status = StatusCode::OK;
status.as_u16();             // 200
status.is_success();         // true (2xx)
status.is_client_error();    // false (4xx)
status.is_server_error();    // false (5xx)
status.as_str();             // "OK"
```

---

## Method

```rust
pub enum Method {
    Get, Post, Put, Patch, Delete, Head, Options, Custom(String),
}
```

```rust
let method = Method::Get;
method.as_str();     // "GET"
Method::parse("POST");  // Method::Post
```

---

## Headers

```rust
pub struct Headers { h: HashMap<String, String> }
```

```rust
let mut h = Headers::new();
h.set("Content-Type", "application/json");
h.get("content-type");          // Some("application/json")
h.has("Content-Type");          // true
h.content_length();             // Option<u64>
h.content_type();               // Option<&str>

for (k, v) in h.iter() {
    println!("{}: {}", k, v);
}
```

Headers are case-insensitive — stored keys are lowercased.

---

## Uri

```rust
pub struct Uri {
    pub scheme: String,
    pub host: String,
    pub port: u16,
    pub path: String,
    pub query: HashMap<String, String>,
    pub fragment: Option<String>,
}
```

```rust
let uri = Uri::parse("https://example.com:8080/path?page=1#top").unwrap();
uri.scheme;    // "https"
uri.host;      // "example.com"
uri.port;      // 8080
uri.path;      // "/path"
uri.query.get("page");  // Some("1")
uri.fragment;  // Some("top")
```

---

## Request

```rust
pub struct Request {
    pub method: Method,
    pub uri: Uri,
    pub headers: Headers,
    pub body: Vec<u8>,
    pub params: HashMap<String, String>,
}
```

### Methods

```rust
fn handler(req: Request) -> Response {
    req.method();                // Method::Get
    req.uri.path;               // "/users/:id"
    req.param("id");            // Some("42") — route parameter
    req.query("page");          // Some("2") — query string
    req.header("Content-Type"); // Option<&str>
    req.body_str();             // &str
    req.cookie("session");      // Option<String>
    req.cookies();              // HashMap<String, String>
    req.json::<MyType>();       // Result<MyType, String>
}
```

---

## Response

```rust
pub struct Response {
    pub status: StatusCode,
    pub headers: Headers,
    pub body: Vec<u8>,
}
```

### Constructors

```rust
Response::new(StatusCode::OK);
Response::ok();                       // 200
Response::not_found();                // 404
Response::html("<h1>Hello</h1>");     // text/html
Response::text("plain");              // text/plain
Response::json(&my_struct);           // application/json — returns Response directly
```

### Builder Methods

```rust
Response::html("content")
    .status(StatusCode::CREATED)
    .with_header("X-Custom", "value")
    .cookie(Cookie::new("session", "abc"))
    .remove_cookie("old_cookie")
    .with_body(additional_bytes);
```

### Raw

```rust
let bytes: Vec<u8> = response.to_raw();
// HTTP/1.1 200 OK\r\ncontent-type: text/html\r\n...
```

---

## Cookie

```rust
pub struct Cookie {
    pub name: String,
    pub value: String,
    pub expires: Option<u64>,
    pub path: Option<String>,
    pub domain: Option<String>,
    pub secure: bool,
    pub http_only: bool,
    pub same_site: Option<String>,
}
```

### Constructors

```rust
Cookie::new("name", "value");         // basic, HttpOnly, SameSite=Lax
Cookie::forever("theme", "dark");     // 5 year expiry
Cookie::forgotten("old_cookie");      // Max-Age=0
```

### Builder Methods

```rust
Cookie::new("session", "abc")
    .without_http_only()       // allow JS access
    .with_secure()             // HTTPS only
    .with_path("/admin")
    .with_domain("example.com")
    .with_same_site("Strict")
    .expires_in(3600);         // 1 hour
```

### Output

```rust
let c = Cookie::new("session", "abc");
c.to_header_string();
// "session=abc; Path=/; HttpOnly; SameSite=Lax"
```