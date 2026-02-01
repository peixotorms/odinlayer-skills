---
name: rust-cli
description: Use when building command-line tools or TUI apps in Rust with clap, ratatui, or crossterm. Covers arguments, flags, subcommands, env vars, config files, TOML parsing, colored output with dialoguer and indicatif, terminal control, piping, stdin, stdout, stderr separation, config precedence, exit codes, progress bars, and interactive prompts.
---

# CLI Development

## Domain Constraints

| Domain Rule | Design Constraint | Rust Implication |
|-------------|-------------------|------------------|
| User ergonomics | Clear help, errors | clap derive macros |
| Config precedence | CLI > env > file | Layered config loading |
| Exit codes | Non-zero on error | Proper Result handling |
| Stdout/stderr | Data vs errors | eprintln! for errors |
| Interruptible | Handle Ctrl+C | Signal handling |

## Critical Rules

- **Errors to stderr, data to stdout** — enables piping and scriptability.
- **CLI args > env vars > config file > defaults** — standard override chain.
- **Return non-zero on any error** — script integration depends on exit codes.

## Key Crates

| Purpose | Crate |
|---------|-------|
| Argument parsing | clap |
| Interactive prompts | dialoguer |
| Progress bars | indicatif |
| Colored output | colored |
| Terminal UI | ratatui |
| Terminal control | crossterm |
| Console utilities | console |

## CLI Structure Pattern

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "myapp", about = "My CLI tool")]
struct Cli {
    /// Enable verbose output
    #[arg(short, long)]
    verbose: bool,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Initialize a new project
    Init { name: String },
    /// Run the application
    Run {
        #[arg(short, long)]
        port: Option<u16>,
    },
}

fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    match cli.command {
        Commands::Init { name } => init_project(&name)?,
        Commands::Run { port } => run_server(port.unwrap_or(8080))?,
    }
    Ok(())
}
```

## Common Mistakes

| Mistake | Domain Violation | Fix |
|---------|-----------------|-----|
| Errors to stdout | Breaks piping | eprintln! |
| No help text | Poor UX | #[arg(help = "...")] |
| Panic on error | Bad exit code | Result + proper handling |
| No progress for long ops | User uncertainty | indicatif |
