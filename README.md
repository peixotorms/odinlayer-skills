# AI Skills Marketplace

A curated collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugins for development best practices and coding standards.

## Quick Start

```bash
# 1. Add this marketplace (one-time setup)
claude plugin marketplace add peixotorms/ai-skills

# 2. Install the plugins you want
claude plugin install rust-skills
```

## Available Plugins

### rust-skills

**Microsoft Pragmatic Rust Guidelines** — 48 production rules for writing high-quality Rust code.

```bash
claude plugin install rust-skills
```

**What's included:**

| Component | Type | Description |
|-----------|------|-------------|
| `rust-guidelines` | Skill | Auto-activates when writing or reviewing Rust code. Quick-reference tables for all 48 guidelines covering error handling, safety, API design, documentation, naming, performance, library resilience, and logging. |
| `/rust-review` | Command | Manually review any Rust file against the guidelines. Usage: `/rust-review src/main.rs` |

**Guidelines covered:**

- **Error Handling** — `anyhow`/`eyre` for apps, canonical error structs for libs, panics only for bugs
- **Safety** — `unsafe` only with valid reason (novel abstraction, perf, FFI), all code must be sound
- **API Design** — `impl AsRef`, no smart pointers in public APIs, concrete > generics > `dyn Trait`, builders
- **Documentation** — Summary < 15 words, canonical sections, module-level docs
- **Naming** — No weasel words, magic values documented, `#[expect]` over `#[allow]`
- **Performance** — mimalloc, profiling with criterion/divan, yield points in async
- **Library Resilience** — Avoid statics, mockable I/O, no glob re-exports, additive features, `Send` types
- **Logging** — Structured logging, named events, OTel conventions, redact sensitive data
- **Static Verification** — Recommended compiler and clippy lint configuration

Based on [Microsoft Pragmatic Rust Guidelines](https://microsoft.github.io/rust-guidelines/).

## Managing Plugins

```bash
# Update all marketplace plugins
claude plugin marketplace update ai-skills

# List installed plugins
claude plugin list

# Disable/enable a plugin temporarily
claude plugin disable rust-skills
claude plugin enable rust-skills

# Uninstall
claude plugin uninstall rust-skills

# Remove the marketplace entirely
claude plugin marketplace remove ai-skills
```

## Contributing

To add a new plugin to this marketplace:

1. Create a directory under `plugins/your-plugin-name/`
2. Add `.claude-plugin/plugin.json` with name, description, version, author
3. Add skills in `skills/`, commands in `commands/`, or other components
4. Add an entry to `.claude-plugin/marketplace.json`
5. Submit a PR

See the [Claude Code plugin docs](https://docs.anthropic.com/en/docs/claude-code/plugins) for the full plugin specification.
