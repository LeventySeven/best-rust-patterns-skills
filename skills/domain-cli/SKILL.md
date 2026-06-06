---
name: domain-cli
description: "Use when building CLI tools. Keywords: CLI, command line, terminal, clap, structopt, argument parsing, subcommand, interactive, TUI, ratatui, crossterm, indicatif, progress bar, colored output, shell completion, config file, environment variable, 命令行, 终端应用, 参数解析"
globs: ["**/Cargo.toml"]
user-invocable: false
---

# CLI Domain

> **Layer 3: Domain Constraints**

## Domain Constraints → Design Implications

| Domain Rule | Design Constraint | Rust Implication |
|-------------|-------------------|------------------|
| User ergonomics | Clear help, errors | clap derive macros |
| Config precedence | CLI > env > file | Layered config loading |
| Exit codes | Non-zero on error | Proper Result handling |
| Stdout/stderr | Data vs errors | eprintln! for errors |
| Interruptible | Handle Ctrl+C | Signal handling |

---

## Critical Constraints

### User Communication

```
RULE: Errors to stderr, data to stdout
WHY: Pipeable output, scriptability
RUST: eprintln! for errors, println! for data
```

### Configuration Priority

```
RULE: CLI args > env vars > config file > defaults
WHY: User expectation, override capability
RUST: Layered config with clap + figment/config
```

### Exit Codes

```
RULE: Return non-zero on any error
WHY: Script integration, automation
RUST: main() -> Result<(), Error> or explicit exit()
```

---

## Spawning Child Processes (orphan-free)

A CLI that spawns subprocesses MUST guarantee the children die when it does — otherwise a `Ctrl+C` or crash leaves orphaned processes holding ports/files. The `Interruptible` constraint above is incomplete without this.

```
RULE: Every spawned child dies with the parent
WHY: Ctrl+C / crash must not leak orphan processes
RUST: kill_on_drop(true) everywhere; on Linux also PR_SET_PDEATHSIG
```

Production discipline (OpenAI Codex, `codex-rs/core/src/spawn.rs`). The source documents the rule as comments (verbatim):

```text
// On Linux: prctl(PR_SET_PDEATHSIG, SIGTERM) ensures child dies when Codex dies (even SIGKILL)
// On all platforms: kill_on_drop(true) on tokio::process::Command
// For shell tools: stdin = Stdio::null() (prevents ripgrep-style stdin hangs)
//                  stdout/stderr = Stdio::piped()
```

Applied as two layered guards (illustrative skeleton — the discipline those comments mandate):

```rust
let mut cmd = tokio::process::Command::new(program);
cmd.stdin(Stdio::null())     // shell tools: avoid ripgrep-style stdin hangs
   .stdout(Stdio::piped())
   .kill_on_drop(true);      // secondary guard (all platforms): die if the handle drops
// primary guard (Linux): set prctl(PR_SET_PDEATHSIG, SIGTERM) in a pre_exec hook,
// so the child dies even when the parent is SIGKILL'd (drop never runs then)
```

`kill_on_drop` alone is not enough: if the parent is `SIGKILL`'d, no destructor runs, so on Linux you also need `prctl(PR_SET_PDEATHSIG, SIGTERM)` set in a `pre_exec` hook on the child.

### Killing a child while you stream its output

If your CLI reads a long-running child's stdout on one thread/task and wants to cancel it from another (the Ctrl+C handler), plain `std::process::Child` is a footgun: `wait()` takes `&mut self`, so the reading thread holds the only mutable handle and the canceller has nothing to call `kill()` on — and racing `kill`/reap on raw handles risks killing a reused PID.

Fix (Tauri shell plugin, `process/mod.rs`): wrap in `shared_child::SharedChild`, which is `Send + Sync` behind an `Arc` — one task calls `wait()` while another calls `kill()` safely.

```rust
let child = Arc::new(SharedChild::spawn(&mut command)?);
let waiter = child.clone();
std::thread::spawn(move || { let _ = waiter.wait(); }); // reaper task
// Ctrl+C handler, on another thread:
child.kill()?;   // safe: SharedChild is Send + Sync
```

Pipe the child's stdout/stderr through `os_pipe` + a bounded `tokio::sync::mpsc::channel(1)` so output backpressures instead of buffering unbounded:

```rust
// (Tauri — plugins/shell/src/process/mod.rs)
let (stdout_reader, stdout_writer) = pipe()?;
let (stderr_reader, stderr_writer) = pipe()?;
command.stdout(stdout_writer);
command.stderr(stderr_writer);

let child = Arc::new(SharedChild::spawn(&mut command)?);
let (tx, rx) = channel(1);   // bounded(1): output backpressures
spawn_pipe_reader(tx.clone(), guard.clone(), stdout_reader, CommandEvent::Stdout, raw);
spawn_pipe_reader(tx.clone(), guard.clone(), stderr_reader, CommandEvent::Stderr, raw);
```

---

## Trace Down ↓

From constraints to design (Layer 2):

```
"Need argument parsing"
    ↓ m05-type-driven: Derive structs for args
    ↓ clap: #[derive(Parser)]

"Need config layering"
    ↓ m09-domain: Config as domain object
    ↓ figment/config: Layer sources

"Need progress display"
    ↓ m12-lifecycle: Progress bar as RAII
    ↓ indicatif: ProgressBar
```

---

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

## Design Patterns

| Pattern | Purpose | Implementation |
|---------|---------|----------------|
| Args struct | Type-safe args | `#[derive(Parser)]` |
| Subcommands | Command hierarchy | `#[derive(Subcommand)]` |
| Config layers | Override precedence | CLI > env > file |
| Progress | User feedback | `ProgressBar::new(len)` |

## Code Pattern: CLI Structure

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

---

## Common Mistakes

| Mistake | Domain Violation | Fix |
|---------|-----------------|-----|
| Errors to stdout | Breaks piping | eprintln! |
| No help text | Poor UX | #[arg(help = "...")] |
| Panic on error | Bad exit code | Result + proper handling |
| No progress for long ops | User uncertainty | indicatif |

---

## Trace to Layer 1

| Constraint | Layer 2 Pattern | Layer 1 Implementation |
|------------|-----------------|------------------------|
| Type-safe args | Derive macros | clap Parser |
| Error handling | Result propagation | anyhow + exit codes |
| User feedback | Progress RAII | indicatif ProgressBar |
| Config precedence | Builder pattern | Layered sources |

---

## Related Skills

| When | See |
|------|-----|
| Error handling | m06-error-handling |
| Type-driven args | m05-type-driven |
| Progress lifecycle | m12-lifecycle |
| Async CLI | m07-concurrency |
