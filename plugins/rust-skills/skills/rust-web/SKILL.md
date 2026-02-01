---
name: rust-web
description: Use when building web services, REST APIs, or HTTP servers in Rust with axum, actix-web, warp, or rocket. Covers tower middleware, hyper, reqwest HTTP client, router design, JSON handling, REST and OpenAPI, CORS configuration, WebSocket, SSE (server-sent events), graceful shutdown, state sharing, handler patterns, extractors, IntoResponse error handling, and framework comparison.
---

# Web Development

## Domain Constraints

| Domain Rule | Design Constraint | Rust Implication |
|-------------|-------------------|------------------|
| Stateless HTTP | No request-local globals | State in extractors |
| Concurrency | Handle many connections | Async, Send + Sync |
| Latency SLA | Fast response | Efficient ownership |
| Security | Input validation | Type-safe extractors |
| Observability | Request tracing | tracing + tower layers |

## Critical Rules

- **Web handlers must not block** — block one task = block many requests. Use `spawn_blocking` for CPU work.
- **Shared state must be thread-safe** — handlers run on any thread. Use `Arc<T>`, `Arc<RwLock<T>>` for mutable.
- **Resources live only for request duration** — use extractors for proper ownership.

## Framework Comparison

| Framework | Style | Best For |
|-----------|-------|----------|
| axum | Functional, tower | Modern APIs |
| actix-web | Actor-based | High performance |
| warp | Filter composition | Composable APIs |
| rocket | Macro-driven | Rapid development |

## Key Crates

| Purpose | Crate |
|---------|-------|
| HTTP server | axum, actix-web |
| HTTP client | reqwest |
| JSON | serde_json |
| Auth/JWT | jsonwebtoken |
| Session | tower-sessions |
| Database | sqlx, diesel |
| Middleware | tower |

## Design Patterns

| Pattern | Purpose | Implementation |
|---------|---------|----------------|
| Extractors | Request parsing | `State(db)`, `Json(payload)` |
| Error response | Unified errors | `impl IntoResponse` |
| Middleware | Cross-cutting | Tower layers |
| Shared state | App config | `Arc<AppState>` |

## Axum Handler Pattern

```rust
async fn handler(
    State(db): State<Arc<DbPool>>,
    Json(payload): Json<CreateUser>,
) -> Result<Json<User>, AppError> {
    let user = db.create_user(&payload).await?;
    Ok(Json(user))
}

// Error handling
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            Self::NotFound => (StatusCode::NOT_FOUND, "Not found"),
            Self::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "Internal error"),
        };
        (status, Json(json!({"error": message}))).into_response()
    }
}
```

## Common Mistakes

| Mistake | Domain Violation | Fix |
|---------|-----------------|-----|
| Blocking in handler | Latency spike | spawn_blocking |
| Rc in state | Not Send + Sync | Use Arc |
| No validation | Security risk | Type-safe extractors |
| No error response | Bad UX | IntoResponse impl |
