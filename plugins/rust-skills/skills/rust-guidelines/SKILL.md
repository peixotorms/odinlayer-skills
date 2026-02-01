---
name: rust-guidelines
description: Use when writing, reviewing, or refactoring Rust code. Covers error handling, API design, unsafe usage, safety invariants, ownership patterns, lifetimes, concurrency, async, anti-patterns, performance optimization, documentation, naming, logging, testing, and library resilience.
---

# Rust Guidelines

## Overview

Comprehensive Rust guidelines combining Microsoft's Pragmatic Rust Guidelines (48 production rules) with community patterns for ownership, concurrency, unsafe code, and anti-pattern avoidance.

Full Microsoft reference: @microsoft-rust-guidelines.md

## Error Handling

| Context | Pattern | Guideline |
|---------|---------|-----------|
| Applications | Use `anyhow`/`eyre` for convenience | M-APP-ERROR |
| Libraries | Canonical error structs with `Backtrace` | M-ERRORS-CANONICAL-STRUCTS |
| Bugs detected | Panic, not `Result` | M-PANIC-ON-BUG |
| Panics | Mean "stop the program", never for control flow | M-PANIC-IS-STOP |
| Library code | Never `panic!` — return `Result` | Anti-pattern #6 |
| Silent failure | Never `let _ = file.write_all(data)` — propagate or log | Anti-pattern #5 |

```rust
// BAD: library panics on bad input
pub fn parse(input: &str) -> Data {
    if input.is_empty() { panic!("empty"); }
    // ...
}

// GOOD: return Result
pub fn parse(input: &str) -> Result<Data, ParseError> {
    if input.is_empty() { return Err(ParseError::EmptyInput); }
    // ...
}
```

## Safety & Unsafe

| Rule | Guideline |
|------|-----------|
| `unsafe` only for UB risk (not "dangerous") | M-UNSAFE-IMPLIES-UB |
| Must have valid reason: novel abstraction, perf, FFI | M-UNSAFE |
| All code must be sound — no exceptions | M-UNSOUND |
| Must pass Miri, include safety comments | M-UNSAFE |

### Before Writing Unsafe — Checklist

1. **Do you really need unsafe?** Try safe alternatives first: restructure for borrow checker, use `Cell`/`RefCell`/`Mutex`, check for a safe crate
2. **Identify the operation**: pointer deref, `unsafe fn` call, mutable static, unsafe trait impl, union field, FFI
3. **Document invariants** with `// SAFETY:` comments explaining what must hold and why it does
4. **Test with Miri**: `cargo miri test`

### Top 10 Unsafe Pitfalls

| # | Pitfall | Detection | Fix |
|---|---------|-----------|-----|
| 1 | Dangling pointer from local | Miri | Heap-allocate or return value |
| 2 | CString dropped, pointer dangling | Miri | Caller keeps CString alive, or `into_raw` |
| 3 | `Vec::set_len` with uninitialized data | Miri | Use `push`/`resize` instead |
| 4 | Reference to `#[repr(packed)]` field | Miri, UBsan | `read_unaligned` via `addr_of!` |
| 5 | Mutable aliasing through raw pointers | Miri | Use single pointer, sequential access |
| 6 | `transmute` to wrong size | Compile error/Miri | Use `as` conversion |
| 7 | Invalid enum discriminant via transmute | Manual review | Use `match` or `TryFrom` |
| 8 | Panic unwinding across FFI boundary | Testing | Wrap with `catch_unwind` |
| 9 | Double free from `Clone` + `Drop` on raw ptr | ASan | Don't impl Clone, or use refcounting |
| 10 | `mem::forget` prevents destructor (lock leak) | Manual review | Let guard drop naturally |

```rust
// SAFETY comment examples:
// SAFETY: We checked that index < len above, so this is in bounds.
// SAFETY: The pointer was created from a valid reference and hasn't been invalidated.
// SAFETY: We hold the lock, guaranteeing exclusive access.
// SAFETY: The type is #[repr(C)] and all fields are initialized.
```

## API Design

| Rule | Guideline |
|------|-----------|
| Accept `impl AsRef<str/Path/[u8]>` in functions | M-IMPL-ASREF |
| Accept `impl Read/Write` for I/O (sans-io) | M-IMPL-IO |
| Avoid `Arc/Rc/Box/RefCell` in public APIs | M-AVOID-WRAPPERS |
| Prefer concrete types > generics > `dyn Trait` | M-DI-HIERARCHY |
| Use `PathBuf`/`Path` not `String` for filesystem | M-STRONG-TYPES |
| Essential functionality as inherent methods, not just traits | M-ESSENTIAL-FN-INHERENT |
| Builders for 3+ optional params | M-INIT-BUILDER |
| Accept `&str` not `String` when you don't need ownership | Anti-pattern #7 |
| Borrow (`&Config`) when you only read, don't take ownership | Anti-pattern #19 |

### Type-Driven Design

```rust
// BAD: stringly typed — wrong order compiles fine
fn connect(host: &str, port: &str, timeout: &str) { ... }
connect("8080", "localhost", "30"); // swapped!

// GOOD: strong types prevent misuse at compile time
struct Host(String);
struct Port(u16);
struct Timeout(Duration);
fn connect(host: Host, port: Port, timeout: Timeout) { ... }
```

```rust
// BAD: boolean parameters — unclear at call site
fn fetch(url: &str, use_cache: bool, validate_ssl: bool) { ... }
fetch("https://...", true, false); // what do true/false mean?

// GOOD: options struct
struct FetchOptions { use_cache: bool, validate_ssl: bool }
fn fetch(url: &str, options: FetchOptions) { ... }
```

```rust
// BAD: nested Option — ambiguous semantics
fn find(id: u32) -> Option<Option<User>> { ... } // None vs Some(None)?

// GOOD: explicit enum
enum FindResult { Found(User), NotFound, Error(String) }
```

## Ownership & Lifetimes

### Key Anti-Patterns

```rust
// BAD: clone to dodge borrow checker
for item in data.clone() { println!("{}", item); }
use_data(data);

// GOOD: borrow when you don't need ownership
for item in &data { println!("{}", item); }
use_data(data);
```

```rust
// BAD: holding reference prevents mutation
let mut data = vec![1, 2, 3];
let first = &data[0];
data.push(4); // ERROR: data is borrowed

// GOOD: copy the value out
let first = data[0]; // i32 is Copy
data.push(4); // OK
```

### Lifetime Elision Rules

The compiler infers lifetimes in this order:
1. Each reference parameter gets its own lifetime
2. If exactly one input lifetime, it applies to all outputs
3. If `&self`/`&mut self` exists, its lifetime applies to all outputs

When elision doesn't work, annotate explicitly. Common pattern:

```rust
struct Parser<'input> {
    source: &'input str, // Parser cannot outlive its input
}
```

### Interior Mutability Decision Table

| Need | Type | Thread-safe? |
|------|------|-------------|
| Simple value mutation through `&` | `Cell<T>` | No |
| Complex value, runtime borrow check | `RefCell<T>` | No |
| Shared mutable state across threads | `Arc<Mutex<T>>` | Yes |
| Read-heavy shared state | `Arc<RwLock<T>>` | Yes |
| Simple counter/flag across threads | `AtomicU32`, `AtomicBool` | Yes |
| One-time initialization | `OnceLock<T>` | Yes |

## Concurrency & Async

### Thread Safety

| Error | Cause | Fix |
|-------|-------|-----|
| `Rc<T>` cannot be sent between threads | `Rc` is not `Send` | Use `Arc` |
| `RefCell<T>` is not `Sync` | Not thread-safe | Use `Mutex` or `RwLock` |
| Cannot mutate through `Arc` | No interior mutability | `Arc<Mutex<T>>` or `Arc<AtomicT>` |

### Deadlock Prevention

- **Lock ordering**: Always acquire multiple locks in the same order across all threads
- **Self-deadlock**: Never lock the same `std::Mutex` twice (use `parking_lot::ReentrantMutex` if needed)
- **Scope locks tightly**: Drop guards as soon as possible

### Async Patterns

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

### Critical Async Mistakes

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

### Channel Disconnection

```rust
// Always handle disconnection
for msg in rx {
    handle(msg);
} // loop ends when all senders drop

// Or explicit matching
match rx.try_recv() {
    Ok(msg) => handle(msg),
    Err(TryRecvError::Empty) => continue,
    Err(TryRecvError::Disconnected) => break,
}
```

## Performance

| Rule | Guideline |
|------|-----------|
| Use mimalloc as global allocator in apps | M-MIMALLOC-APPS |
| Profile hot paths early with criterion/divan | M-HOTPATH |
| Optimize items/CPU-cycle, avoid empty cycles | M-THROUGHPUT |
| `yield_now().await` every 10-100us in CPU loops | M-YIELD-POINTS |

### Allocation Avoidance

```rust
// Use Cow to avoid allocation when not needed
use std::borrow::Cow;

fn to_uppercase(s: &str) -> Cow<'_, str> {
    if s.chars().all(|c| c.is_uppercase()) {
        Cow::Borrowed(s) // no allocation
    } else {
        Cow::Owned(s.to_uppercase())
    }
}
```

```rust
// Reuse buffers across iterations
let mut buffer = Vec::new();
for item in items {
    buffer.clear(); // reuse allocation
    process(&mut buffer, item);
}
```

### Collection Choice

| Need | Collection | Notes |
|------|------------|-------|
| Sequential access | `Vec<T>` | Best cache locality |
| Random access by key | `HashMap<K, V>` | O(1) lookup |
| Ordered keys | `BTreeMap<K, V>` | O(log n) lookup |
| Small sets (<20) | `Vec<T>` + linear search | Lower overhead |
| FIFO queue | `VecDeque<T>` | O(1) push/pop both ends |
| Frequent membership check | `HashSet<T>` | O(1) vs O(n) for Vec |

### Iterator Best Practices

```rust
// BAD: bounds checking on each access
for i in 0..vec.len() { process(vec[i]); }

// GOOD: no bounds checking, idiomatic
for item in &vec { process(item); }

// BAD: unnecessary intermediate allocation
let filtered: Vec<_> = items.iter().filter(|x| x.valid).collect();
for item in filtered { process(item); }

// GOOD: chain iterators — no allocation
for item in items.iter().filter(|x| x.valid) { process(item); }

// Counting? Don't collect.
let count = items.iter().filter(|x| x.valid).count();
```

### String Optimization

```rust
// BAD: O(n^2) allocations in loop
let mut result = String::new();
for s in strings { result = result + &s; }

// GOOD: pre-calculate capacity
let total_len: usize = strings.iter().map(|s| s.len()).sum();
let mut result = String::with_capacity(total_len);
for s in strings { result.push_str(&s); }

// BEST: use join for simple concatenation
let result = strings.join("");
```

### Memory Layout

```rust
// BAD: 24 bytes due to padding
struct Bad { a: u8, b: u64, c: u8 } // 1+7pad + 8 + 1+7pad

// GOOD: 16 bytes — order fields largest-first
struct Good { b: u64, a: u8, c: u8 } // 8 + 1 + 1 + 6pad

// Box large enum variants
enum Message {
    Quit,
    Data(Box<[u8; 10000]>), // pointer-sized, not 10KB
}
```

### Release Build Settings

```toml
[profile.release]
lto = true           # Link-time optimization
codegen-units = 1    # Slower compile, faster code
panic = "abort"      # Smaller binary, no unwinding overhead
strip = true         # Strip debug symbols
```

## Documentation

| Rule | Guideline |
|------|-----------|
| Summary sentence < 15 words | M-FIRST-DOC-SENTENCE |
| Canonical sections: Examples, Errors, Panics, Safety | M-CANONICAL-DOCS |
| Module-level `//!` docs required | M-MODULE-DOCS |
| No parameter tables — describe in prose | M-CANONICAL-DOCS |
| `#[doc(inline)]` on `pub use` re-exports | M-DOC-INLINE |

## Naming & Style

| Rule | Guideline |
|------|-----------|
| No weasel words (Service, Manager, Factory) | M-CONCISE-NAMES |
| Use `Builder` not `Factory` | M-CONCISE-NAMES |
| Magic values need comments explaining *why* | M-DOCUMENTED-MAGIC |
| Use `#[expect]` not `#[allow]` for lint overrides | M-LINT-OVERRIDE-EXPECT |

## Library Resilience

| Rule | Guideline |
|------|-----------|
| Avoid `static` if consistent view matters | M-AVOID-STATICS |
| I/O and syscalls must be mockable | M-MOCKABLE-SYSCALLS |
| No glob re-exports (`pub use foo::*`) | M-NO-GLOB-REEXPORTS |
| Test utilities behind `test-util` feature | M-TEST-UTIL |
| Features must be additive | M-FEATURES-ADDITIVE |
| Don't leak external types in public APIs | M-DONT-LEAK-TYPES |
| Types should be `Send` | M-TYPES-SEND |
| Split crates when in doubt | M-SMALLER-CRATES |

## Logging

| Rule | Guideline |
|------|-----------|
| Structured logging with message templates | M-LOG-STRUCTURED |
| Named events: `component.operation.state` | M-LOG-STRUCTURED |
| OTel semantic conventions for attributes | M-LOG-STRUCTURED |
| Redact sensitive data | M-LOG-STRUCTURED |

## Static Verification

Recommended `Cargo.toml` lints:

```toml
[lints.rust]
missing_debug_implementations = "warn"
redundant_imports = "warn"
unsafe_op_in_unsafe_fn = "warn"
unused_lifetimes = "warn"

[lints.clippy]
cargo = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
correctness = { level = "warn", priority = -1 }
perf = { level = "warn", priority = -1 }
style = { level = "warn", priority = -1 }
suspicious = { level = "warn", priority = -1 }
undocumented_unsafe_blocks = "warn"
```

## Anti-Pattern Quick Reference

| Anti-Pattern | Better Alternative |
|--------------|--------------------|
| Clone to satisfy borrow checker | Borrow with `&` |
| `unwrap()` everywhere | Propagate with `?` |
| `String` parameters | `&str` parameters |
| Index loops `vec[i]` | Iterator loops `for item in &vec` |
| Collect then process | Chain iterators |
| `Mutex` for read-heavy data | `RwLock` |
| Lock held across `.await` | Scope the lock |
| `std::thread::sleep` in async | `tokio::time::sleep` |
| Stringly typed APIs | Newtype wrappers |
| Boolean parameters | Options struct or enum |
| `Box<T>` returns unnecessarily | Return `T` directly |
| Macro for simple operations | Plain function |
| `Option<Option<T>>` | Custom enum with clear semantics |
| `pub use foo::*` | Explicit re-exports |
| `#[allow(clippy::...)]` | `#[expect(clippy::...)]` |
| `transmute` for enum casts | `match` or `TryFrom` |
| `Arc<Mutex<T>>` in public API | Clean interface hiding internals |
| `String` for file paths | `PathBuf` / `Path` |
| `static` items across crate versions | Config struct passed at init |

## FFI Safety

```rust
// BAD: panic unwinding across FFI is UB
#[no_mangle]
extern "C" fn callback(x: i32) -> i32 {
    if x < 0 { panic!("negative!"); } // UB!
    x * 2
}

// GOOD: catch panics at FFI boundary
#[no_mangle]
extern "C" fn callback(x: i32) -> i32 {
    std::panic::catch_unwind(|| {
        if x < 0 { panic!("negative!"); }
        x * 2
    }).unwrap_or(-1) // error code on panic
}
```

## Thread Panic Handling

```rust
// Always handle thread panics
let handle = std::thread::spawn(|| {
    risky_operation()
});

match handle.join() {
    Ok(result) => use_result(result),
    Err(e) => eprintln!("Thread panicked: {:?}", e),
}
```
