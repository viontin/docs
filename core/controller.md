# Controller
> Last updated: 2026-05-25

**Module:** `viontin_framework::controller` (requires `orm` feature)  
**Trait:** `Controller<M: Entity + Serialize>`  
**Default:** `DefaultController<M, S>`

A **Controller** handles HTTP requests, delegates to a `Service`, and returns HTTP responses. It provides default RESTful actions (`index`, `show`, `store`, `update`, `destroy`) with `before`/`after` lifecycle hooks.

---

## Philosophy

- **Thin controllers** — keep HTTP handling here, business logic in Services.
- **DI-first** — receives a `Service` via constructor injection.
- **Hook-based** — `before()` for auth/validation, `after()` for logging/headers.
- **Optional** — use route handlers directly via `boot().get(...)` if you prefer.

---

## Trait Definition

```rust
pub trait Controller<M: Entity + Serialize + 'static>: Debug + Send + Sync {
    fn service(&self) -> &dyn Service<M>;
    fn resource_name(&self) -> &str;

    // Hooks
    fn before(&self, _req: &Request, _action: &str) -> Result<(), Response> { Ok(()) }
    fn after(&self, _req: &Request, _res: &mut Response, _action: &str) {}

    // Default RESTful actions
    fn index(&self, req: Request) -> Response;
    fn show(&self, req: Request) -> Response;
    fn store(&self, req: Request) -> Response;
    fn update(&self, req: Request) -> Response;
    fn destroy(&self, req: Request) -> Response;
}
```

Each default action:
1. Calls `before()` — short-circuits on error
2. Executes the action (delegates to `Service`)
3. Calls `after()` for post-processing

---

## Usage

### With DefaultController (Zero Boilerplate)

```rust
use viontin_framework::controller::DefaultController;
use viontin_framework::service::DefaultService;

type UserService = DefaultService<User, UserRepo>;
type UserController = DefaultController<User, UserService>;

let ctrl = UserController::new(
    UserService::new(UserRepo { conn: Box::new(conn) }),
    "users",
);

// Register routes manually:
boot()
    .get("/users", {
        let c = Arc::new(ctrl);
        Arc::new(move |req| c.index(req))
    })
    .serve(":3000");
```

### Custom Controller with Hooks

```rust
use viontin_framework::controller::Controller;
use viontin::fw::http::{Request, Response, StatusCode};

pub struct UserController {
    pub service: UserService,
}

impl Controller<User> for UserController {
    fn service(&self) -> &dyn Service<User> { &self.service }
    fn resource_name(&self) -> &str { "users" }

    fn before(&self, req: &Request, action: &str) -> Result<(), Response> {
        // Auth check — only admins can delete
        if action == "destroy" && !is_admin(req) {
            return Err(Response::html("Forbidden").with_status(StatusCode::FORBIDDEN));
        }
        Ok(())
    }

    fn after(&self, req: &Request, res: &mut Response, action: &str) {
        // Log every action
        println!("[audit] {} {} — {}",
            req.method, req.uri.path, res.status);
    }
}
```

### Register RESTful Routes

```rust
fn register_routes(router: Router, ctrl: Arc<UserController>) -> Router {
    let c = ctrl.clone();
    router
        .get("/users", {
            let c = c.clone();
            Arc::new(move |req| c.index(req))
        })
        .get("/users/:id", {
            let c = c.clone();
            Arc::new(move |req| c.show(req))
        })
        .post("/users", {
            let c = c.clone();
            Arc::new(move |req| c.store(req))
        })
        .put("/users/:id", {
            let c = c.clone();
            Arc::new(move |req| c.update(req))
        })
        .delete("/users/:id", {
            let c = c.clone();
            Arc::new(move |req| c.destroy(req))
        })
}
```

---

## Default Actions

| Action | Method | Path | Description |
|--------|--------|------|-------------|
| `index` | GET | `/{resources}` | List all entities |
| `show` | GET | `/{resources}/:id` | Show one entity |
| `store` | POST | `/{resources}` | Create new entity |
| `update` | PUT | `/{resources}/:id` | Update entity |
| `destroy` | DELETE | `/{resources}/:id` | Delete entity |

---

## Scaffolding

```bash
viontin make:controller UserController
```

Creates `src/controllers/user_controller.rs` with a `Controller` implementation template.

---

## See Also

- [Service](../core/service) — business logic layer
- [Repository](repository) — data access layer
- [Entity](entity) — business objects
- [Web App](web-app) — routing, request/response
- [FormRequest / Validation](validation) — request validation