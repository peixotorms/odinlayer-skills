---
name: rust-async
description: Use when writing async or concurrent Rust code with tokio, async-std, futures, channels, mutexes, threads, spawn, or runtime. Covers async patterns, Pin, poll, Stream, timeout, graceful shutdown, task management, blocking in async, mutex across await, select, JoinSet, CancellationToken, sync primitives, broadcast, mpsc, oneshot, watch channels, deadlock prevention, and thread safety.
---

# Concurrency & Async

## Thread Safety

| Error | Cause | Fix |
|-------|-------|-----|
| `Rc<T>` cannot be sent between threads | `Rc` is not `Send` | Use `Arc` |
| `RefCell<T>` is not `Sync` | Not thread-safe | Use `Mutex` or `RwLock` |
| Cannot mutate through `Arc` | No interior mutability | `Arc<Mutex<T>>` or `Arc<AtomicT>` |

## Deadlock Prevention

- **Lock ordering**: Always acquire multiple locks in the same order across all threads
- **Self-deadlock**: Never lock the same `std::Mutex` twice (use `parking_lot::ReentrantMutex` if needed)
- **Scope locks tightly**: Drop guards as soon as possible
- **Atomics for primitives**: Use `AtomicBool`/`AtomicUsize` instead of `Mutex<bool>`
- **Choose memory order carefully**: `Relaxed` / `Acquire` / `Release` / `SeqCst`

## Async Patterns

| Pattern | When to Use |
|---------|-------------|
| `tokio::spawn` | Fire-and-forget concurrent tasks |
| `tokio::select!` | Race multiple futures, cancel losers |
| `mpsc` channel | Fan-in: many producers, one consumer |
| `oneshot` channel | Single request-response |
| `broadcast` channel | One producer, many consumers |
| `watch` channel | Latest-value state sharing |
| `JoinSet` | Manage group of spawned tasks |
| `CancellationToken` | Graceful shutdown signaling |

## Critical Async Mistakes

### Mutex Guard Across Await

```rust
// BAD: std::Mutex guard held across .await — blocks executor
async fn bad() {
    let guard = mutex.lock().unwrap();
    some_async_op().await; // guard held across await!
    println!("{}", *guard);
}

// GOOD: scope the lock, copy what you need
async fn good() {
    let value = {
        let guard = mutex.lock().unwrap();
        *guard // copy value
    }; // guard dropped here
    some_async_op().await;
    println!("{}", value);
}
```

### Blocking in Async Context

```rust
// BAD: blocking call in async context
async fn bad() {
    std::thread::sleep(Duration::from_secs(1)); // blocks executor!
    std::fs::read_to_string("file.txt").unwrap(); // blocks!
}

// GOOD: use async equivalents
async fn good() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    tokio::fs::read_to_string("file.txt").await.unwrap();
}

// For CPU work: spawn_blocking
let result = tokio::task::spawn_blocking(|| heavy_computation()).await.unwrap();
```

### Forgetting to Poll Futures

```rust
// BAD: future not polled — does nothing!
fn process() {
    fetch_data(); // returns Future that's immediately dropped
}

// GOOD: await it
async fn process() {
    let data = fetch_data().await;
}
```

### Key Rule

```rust
// Sync for CPU-bound work, async for I/O-bound work
// Don't hold locks across await points
```

## Channel Disconnection

```rust
// Iterate until all senders drop
for msg in rx {
    handle(msg);
}

// Explicit matching for non-blocking
match rx.try_recv() {
    Ok(msg) => handle(msg),
    Err(TryRecvError::Empty) => continue,
    Err(TryRecvError::Disconnected) => break,
}
```

## Thread Panic Handling

```rust
let handle = std::thread::spawn(|| risky_operation());

match handle.join() {
    Ok(result) => use_result(result),
    Err(e) => eprintln!("Thread panicked: {:?}", e),
}
```

## Graceful Shutdown Pattern

```rust
use tokio_util::sync::CancellationToken;

async fn run(token: CancellationToken) {
    loop {
        tokio::select! {
            _ = token.cancelled() => {
                tracing::info!("shutting down");
                break;
            }
            msg = rx.recv() => {
                if let Some(msg) = msg {
                    handle(msg).await;
                }
            }
        }
    }
}
```
