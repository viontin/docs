# Deployment

> **Status:** This guide covers deploying Viontin applications to production. The framework is experimental (v0.1.0) — deployment patterns will evolve.

---

## Build

Viontin compiles to a single native binary with no runtime dependencies (no JVM, no interpreter, no Node.js).

### Release Build

```bash
cargo build --release
```

The binary is at `target/release/viontin` (CLI) or your application binary.

### Stripping (Reduce Binary Size)

```bash
strip target/release/my-app
# Before: ~50MB  After: ~15MB
```

### Cross-Compilation

For deploying to a different architecture (e.g., build on x86_64, deploy to ARM):

```bash
rustup target add aarch64-unknown-linux-gnu
cargo build --release --target aarch64-unknown-linux-gnu
```

---

## Binary Deployment

The compiled binary contains everything — HTTP server, CLI commands, ORM, migrations, embedded templates. Deploy as a single file.

### Docker

```dockerfile
# Dockerfile — multi-stage build
FROM rust:1.85 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/my-app /usr/local/bin/my-app
COPY --from=builder /app/config /app/config
COPY --from=builder /app/.env /app/.env

WORKDIR /app
EXPOSE 3000
CMD ["my-app"]
```

### docker-compose

```yaml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - APP_ENV=production
      - APP_KEY=${APP_KEY}
    volumes:
      - ./data:/app/data
    restart: unless-stopped
```

---

## Environment Configuration

### Required Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_ENV` | `local`, `staging`, `production` | `local` |
| `APP_KEY` | Encryption key (generate with `AesEncrypter::keygen()`) | — |
| `APP_DEBUG` | Enable debug mode | `false` |
| `APP_PORT` | HTTP server port | `3000` |

### Database

```env
# SQLite (default — file-based)
DATABASE_URL=data/app.db

# PostgreSQL (when using viontin-orm-pg)
# DATABASE_URL=postgres://user:pass@host:5432/db
```

### .env Files

The `.env` file is loaded automatically at boot. In production, use your orchestrator's secret management (Docker secrets, Kubernetes secrets, systemd EnvironmentFile) instead of committing `.env` to the repository.

---

## Database Migrations

Run migrations at startup or as a separate step:

```rust
fn main() {
    boot()
        .entry(|ctx| {
            let conn = /* get connection */;
            let mut migrator = Migrator::new(conn);
            migrator.run(migrations()).ok();
            ctx.serve(":3000");
        })
        .run();
}
```

Or via CLI:

```bash
cargo run -- migrate
```

Or via the Viontin CLI (future):

```bash
viontin migrate
```

Migrations are tracked in the `_migrations` table and run only once.

---

## Process Supervision

### systemd

```ini
[Unit]
Description=My Viontin App
After=network.target

[Service]
Type=simple
User=app
WorkingDirectory=/opt/my-app
ExecStart=/usr/local/bin/my-app
Restart=always
RestartSec=5
Environment=APP_ENV=production
Environment=APP_KEY=...

[Install]
WantedBy=multi-user.target
```

### Docker Compose Health Check

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/healthz"]
  interval: 30s
  timeout: 10s
  retries: 3
```

---

## Reverse Proxy

Viontin's HTTP server is suitable for direct exposure behind a reverse proxy (recommended for production).

### Caddy (Recommended)

```caddyfile
example.com {
    reverse_proxy 127.0.0.1:3000
}
```

### nginx

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## Graceful Shutdown

With the `shutdown` feature (enabled by default), the server handles SIGTERM/SIGINT:

```rust
// Enabled automatically with default features
boot().serve(":3000");
```

The server will:
1. Stop accepting new connections
2. Wait for in-flight requests to complete
3. Exit cleanly

---

## Observability

### Request ID

Add the `RequestId` middleware to correlate log entries:

```rust
use viontin_framework::middleware::RequestId;

boot()
    .middleware(RequestId)
    // ...
```

Every response will include an `X-Request-Id` header.

### Query Logging

Log all database queries:

```rust
use viontin_framework::query_log::set_query_logger;

set_query_logger(|sql, duration_ms| {
    log_debug(format!("[query] {:.1}ms — {}", duration_ms, sql));
});
```

### Metrics (Future)

Planned: Prometheus metrics at `/metrics` endpoint for request count, latency, database query time, and cache hit ratios.

---

## Security Checklist

- [ ] Set `APP_ENV=production`
- [ ] Generate a strong `APP_KEY` (use `AesEncrypter::keygen()`)
- [ ] Set `APP_DEBUG=false`
- [ ] Use HTTPS behind a reverse proxy (Caddy, nginx)
- [ ] Enable `CorsMiddleware` with specific origins, not `*`
- [ ] Use `PanicRecovery` middleware
- [ ] Set up database backup strategy
- [ ] Configure log rotation
- [ ] Use non-root user in Docker

---

## Manual Updates

```bash
# 1. Pull new code
git pull origin main

# 2. Build
cargo build --release

# 3. Run migrations
./target/release/my-app migrate

# 4. Restart server (systemd)
sudo systemctl restart my-app

# Or with Docker:
# docker compose up --build -d
```

---

## See Also

- [Configuration](config) — Environment and JSON config
- [Environment](environment) — `.env` loading and helpers
- [Getting Started](getting-started) — First application
