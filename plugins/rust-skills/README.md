# Rust Skills Plugin

Comprehensive Rust guidelines as 11 focused Claude Code skills, plus a review command.

## What's Included

**Core skills** (auto-activate based on context):

| Skill | Focus |
|-------|-------|
| `rust-guidelines` | Error handling, API design, types, ownership, lifetimes, smart pointers, generics, domain modeling, RAII, docs, naming, lints, ecosystem, mental models, anti-patterns |
| `rust-unsafe` | Unsafe code, FFI bindings, raw pointers, safety invariants, Miri, SAFETY comments |
| `rust-async` | Tokio, channels, mutexes, threads, spawn, select, JoinSet, CancellationToken |
| `rust-performance` | Profiling, allocations, Cow, collections, iterators, Rayon, memory layout, release settings |

**Domain skills** (auto-activate for specific domains):

| Skill | Focus |
|-------|-------|
| `rust-web` | axum/actix handlers, extractors, IntoResponse, middleware, state management |
| `rust-cloud-native` | Graceful shutdown, health checks, 12-factor, OpenTelemetry, gRPC |
| `rust-cli` | clap Parser/Subcommand, config precedence, exit codes, progress bars |
| `rust-fintech` | rust_decimal, currency newtypes, immutable records, checked arithmetic |
| `rust-embedded` | no_std, heapless, Mutex\<RefCell\<Option\<T\>\>\>, RTIC/Embassy, cortex-m |
| `rust-iot` | MQTT rumqttc, store-and-forward, backoff retry, power management |
| `rust-ml` | tract ONNX, OnceLock model singleton, batched inference, candle/burn |

**Command:**

- `/rust-review` — Review Rust code against the guidelines

## Installation

```bash
claude plugin marketplace add peixotorms/odinlayer-skills
claude plugin install rust-skills
```

## Usage

Skills activate automatically when Claude detects relevant Rust work. You can also explicitly review code:

```
/rust-review src/main.rs
```

## Sources

- [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/)
- [ZhangHanDong/rust-skills](https://github.com/ZhangHanDong/rust-skills)
