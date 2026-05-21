> **Experimental Project** — This is an experimental project under active development. Not recommended for production use.

# Building Desktop (Native) Applications

> GUI toolkit — coming soon. Native desktop support is on the roadmap and will be a first-class feature of the platform.

---

Viontin does not yet include a built-in GUI toolkit, but the platform is designed to support desktop applications from day one. All of Viontin's core services — configuration, logging, events, storage, scheduling, validation, and i18n — are framework-agnostic and ready to be used as the backend engine for any desktop application.

---

## Current State

| Feature | Status |
|---------|--------|
| Core platform services (config, log, events, etc.) | Ready |
| HTTP server (embedded in-app API) | Ready |
| CLI integration for desktop tooling | Ready |
| Scheduler for background tasks | Ready |
| GUI toolkit | Planned |

While the GUI toolkit is being built, you can use Viontin's platform services as the foundation for your desktop app, paired with any Rust GUI framework:

```rust
use viontin::prelude::*;

fn main() {
    boot().run(|_| {
        // Viontin services are ready
        let config = Config::new();
        let logger = Logger::new();
        let scheduler = Scheduler::new();

        // Launch your GUI here
        // (gui_framework::run(config, logger))
    });
}
```

---

## Platform Services for Desktop

| Service | Usage in Desktop Apps |
|---------|----------------------|
| Config | Load settings from `config/*.json`, environment-specific overrides |
| .env | Manage API keys, paths, environment variables |
| Logging | Structured logging to stdout or file |
| Events | Pub/sub communication between components |
| Scheduler | Cron-based background tasks, periodic sync |
| Storage | File I/O with `LocalStorage` for user data |
| i18n | JSON-based translations for multi-language UI |
| Validation | Input validation for forms and settings |
| Cache | In-memory or file-based caching |
| Queue | Background job processing (sync driver) |

---

## Architecture Pattern

```
┌──────────────────────────────────────────┐
│            GUI Layer (coming soon)         │
│        Native widgets / WebView           │
├──────────────────────────────────────────┤
│        Viontin Platform Layer             │
│   ┌──────┬──────┬──────┬──────────────┐  │
│   │ Config│ Log  │ Events│ Scheduler   │  │
│   ├──────┼──────┼──────┼──────────────┤  │
│   │ Cache│ i18n │ Auth │ Storage      │  │
│   ├──────┼──────┼──────┼──────────────┤  │
│   │ Queue│ Mail │ Notif│ Validation   │  │
│   └──────┴──────┴──────┴──────────────┘  │
├──────────────────────────────────────────┤
│            Application Logic              │
└──────────────────────────────────────────┘
```

The GUI handles rendering and input. Viontin handles everything else.

---

## Roadmap

The native GUI toolkit will include:

- Native window management (create, resize, close, fullscreen)
- Widget system (buttons, inputs, tables, trees, menus)
- Layout engine (flex, grid, dock)
- Styling (theming, dark/light mode)
- Event loop integration with Viontin's scheduler
- Cross-platform support (Linux, macOS, Windows)
- Accessibility foundations

Status and progress will be tracked in the project roadmap.
