# AI Skills Marketplace

A curated collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugins for development best practices and coding standards.

## Quick Start

```bash
# 1. Add this marketplace (one-time setup)
claude plugin marketplace add peixotorms/odinlayer-skills

# 2. Install the plugins you want
claude plugin install rust-skills
claude plugin install go-skills
claude plugin install php-skills
```

## Available Plugins

### rust-skills

**Comprehensive Rust guidelines** — 11 focused skills covering core guidelines, unsafe/FFI, async/concurrency, performance, and 7 domain-specific areas.

```bash
claude plugin install rust-skills
```

One install gives you all 11 skills. Claude loads only the relevant ones per conversation based on what you're working on.

**Skills included:**

| Skill | Auto-activates when... |
|-------|------------------------|
| `rust-guidelines` | Writing, reviewing, building, or deploying Rust code — error handling, API design, types, ownership, lifetimes, smart pointers, generics, domain modeling, RAII, docs, naming, lints, build/deploy workflow, test placement |
| `rust-unsafe` | Writing unsafe code, FFI bindings, raw pointers, or C interop |
| `rust-async` | Writing async/concurrent code with tokio, channels, mutexes, threads |
| `rust-performance` | Optimizing performance — profiling, allocations, collections, Rayon, memory layout |
| `rust-web` | Building web services with axum, actix-web, warp, or rocket |
| `rust-cloud-native` | Building cloud-native apps, microservices, or Kubernetes deployments |
| `rust-cli` | Building CLI tools or TUI apps with clap, ratatui |
| `rust-fintech` | Building financial, trading, or payment systems |
| `rust-embedded` | Developing embedded/no_std/bare-metal for microcontrollers |
| `rust-iot` | Building IoT applications, sensor networks, or edge devices |
| `rust-ml` | Building ML/AI inference pipelines |

**Command included:**

| Command | Description |
|---------|-------------|
| `/rust-review` | Review any Rust file against the guidelines. Usage: `/rust-review src/main.rs` |

**Sources:**

- [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/) (48 production rules)
- [ZhangHanDong/rust-skills](https://github.com/ZhangHanDong/rust-skills) (community patterns for ownership, concurrency, type-driven design, domain modeling)

---

### go-skills

**Idiomatic Go guidelines** — 3 focused skills covering core idioms, concurrency patterns, and error handling.

```bash
claude plugin install go-skills
```

One install gives you all 3 skills. Claude loads only the relevant ones per conversation.

**Skills included:**

| Skill | Auto-activates when... |
|-------|------------------------|
| `go-guidelines` | Writing, reviewing, building, or deploying Go code — formatting, naming, doc comments, control structures, functions, defer, data types, methods, interfaces, embedding, initialization, functional options, generics, testing, linting, build/deploy workflow, test placement, anti-patterns |
| `go-concurrency` | Writing concurrent Go with goroutines, channels, select, sync primitives, worker pools, fan-out/fan-in, context cancellation |
| `go-errors` | Handling errors — wrapping, `errors.Is`/`errors.As`, sentinel errors, custom types, errors-as-values patterns, panic/recover |

**Command included:**

| Command | Description |
|---------|-------------|
| `/go-review` | Review any Go file against the guidelines. Usage: `/go-review main.go` |

**Sources:**

- [Effective Go](https://go.dev/doc/effective_go)
- [Google Go Style Guide](https://google.github.io/styleguide/go/)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Go Proverbs](https://go-proverbs.github.io/)

---

### php-skills

**Comprehensive PHP guidelines** — 2 focused skills covering modern PHP 8.x patterns, type system, OOP, security, and version migration.

```bash
claude plugin install php-skills
```

One install gives you both skills. Claude loads only the relevant ones per conversation.

**Skills included:**

| Skill | Auto-activates when... |
|-------|------------------------|
| `php-guidelines` | Writing, reviewing, or deploying PHP code — type system, strict types, OOP, enums, modern PHP 8.x patterns, generators, fibers, namespaces, error handling, performance, version migration (8.0-8.5), testing, build/deploy, anti-patterns |
| `php-security` | Handling user input, database queries, file operations, authentication — SQL injection, XSS, CSRF, password hashing, session management, file uploads, cryptography |

**Command included:**

| Command | Description |
|---------|-------------|
| `/php-review` | Review any PHP file against the guidelines. Usage: `/php-review src/Controller/UserController.php` |

**Sources:**

- [PHP Manual](https://www.php.net/manual/en/)
- PHP 8.0-8.5 Migration Guides

## Managing Plugins

```bash
# Update all marketplace plugins
claude plugin marketplace update odinlayer-skills

# List installed plugins
claude plugin list

# Disable/enable a plugin temporarily
claude plugin disable rust-skills
claude plugin enable rust-skills

# Uninstall
claude plugin uninstall rust-skills

# Remove the marketplace entirely
claude plugin marketplace remove odinlayer-skills
```

## Contributing

To add a new plugin to this marketplace:

1. Create a directory under `plugins/your-plugin-name/`
2. Add `.claude-plugin/plugin.json` with name, description, version, author
3. Add skills in `skills/`, commands in `commands/`, or other components
4. Add an entry to `.claude-plugin/marketplace.json`
5. Submit a PR

See the [Claude Code plugin docs](https://docs.anthropic.com/en/docs/claude-code/plugins) for the full plugin specification.
