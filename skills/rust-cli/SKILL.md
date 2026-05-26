---
name: rust-cli
description: |
  Rust CLI development — command-line tools, TUI apps, shell integration. Use this skill when:
  - User builds a CLI tool or binary with Rust
  - User asks about clap, argh, or other argument parsing crates
  - User implements subcommands, flags, or shell completion
  - User asks about stdin/stdout/stderr handling or piping
  - User builds a TUI (Terminal UI) with ratatui or crossterm
  - User implements progress bars, spinners, or colored output
  - User works with config files (toml, yaml, json) in a CLI app
  - User asks about cross-platform terminal handling
  - User packages/distributes a CLI tool (cargo-dist, homebrew, etc.)
  Trigger for any Rust CLI or terminal application question.
---

# Rust CLI Development

Build fast, reliable command-line tools in Rust. The ecosystem is excellent: clap for parsing, ratatui for TUI, tracing for observability.

## Quick Setup

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
anyhow = "1"
serde = { version = "1", features = ["derive"] }
toml = "0.8"

# Output
colored = "2"
indicatif = "0.17"    # Progress bars

# Optional TUI
ratatui = "0.29"
crossterm = "0.28"

[profile.release]
opt-level = "s"  # Smaller binary
lto = true
strip = true     # Strip debug symbols
```

## Argument Parsing with Clap

```rust
use clap::{Parser, Subcommand, Args};

/// My awesome CLI tool
#[derive(Parser)]
#[command(name = "mytool")]
#[command(version, about, long_about = None)]
struct Cli {
    /// Global verbosity flag
    #[arg(short, long, global = true, action = clap::ArgAction::Count)]
    verbose: u8,

    /// Config file path
    #[arg(short, long, default_value = "~/.config/mytool.toml")]
    config: std::path::PathBuf,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Initialize a new project
    Init(InitArgs),
    /// Run the main operation
    Run {
        /// Target name
        name: String,
        /// Dry run — don't actually do anything
        #[arg(long)]
        dry_run: bool,
    },
    /// Show status
    Status,
}

#[derive(Args)]
struct InitArgs {
    /// Directory to initialize in
    path: std::path::PathBuf,
    /// Use a template
    #[arg(short, long, value_enum, default_value = "default")]
    template: Template,
}

#[derive(clap::ValueEnum, Clone)]
enum Template {
    Default,
    Minimal,
    Full,
}

fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();

    let log_level = match cli.verbose {
        0 => tracing::Level::WARN,
        1 => tracing::Level::INFO,
        2 => tracing::Level::DEBUG,
        _ => tracing::Level::TRACE,
    };
    tracing_subscriber::fmt().with_max_level(log_level).init();

    match cli.command {
        Commands::Init(args) => run_init(args),
        Commands::Run { name, dry_run } => run_main(&name, dry_run),
        Commands::Status => run_status(),
    }
}
```

## Config Files

```rust
use serde::{Deserialize, Serialize};
use std::path::PathBuf;

#[derive(Debug, Deserialize, Serialize)]
#[serde(default)]
struct Config {
    pub api_url: String,
    pub timeout_secs: u64,
    pub output_format: OutputFormat,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            api_url: "https://api.example.com".into(),
            timeout_secs: 30,
            output_format: OutputFormat::Text,
        }
    }
}

fn load_config(path: &PathBuf) -> anyhow::Result<Config> {
    if !path.exists() {
        return Ok(Config::default());
    }
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("Failed to read config: {}", path.display()))?;
    let config: Config = toml::from_str(&content)
        .context("Failed to parse config file")?;
    Ok(config)
}

fn save_config(config: &Config, path: &PathBuf) -> anyhow::Result<()> {
    if let Some(parent) = path.parent() {
        std::fs::create_dir_all(parent)?;
    }
    let content = toml::to_string_pretty(config)?;
    std::fs::write(path, content)?;
    Ok(())
}

// Platform-appropriate config directory
fn config_path() -> PathBuf {
    dirs::config_dir()
        .unwrap_or_else(|| PathBuf::from("."))
        .join("mytool")
        .join("config.toml")
}
```

## Stdin / Stdout / Piping

```rust
use std::io::{self, BufRead, Write};

fn main() -> anyhow::Result<()> {
    let stdin = io::stdin();
    let stdout = io::stdout();
    let mut out = io::BufWriter::new(stdout.lock()); // Buffered for performance

    for line in stdin.lock().lines() {
        let line = line?;
        // Process and write
        writeln!(out, "{}", process_line(&line))?;
    }
    Ok(())
}

// Detect if we're in a pipe (no TTY)
fn is_piped() -> bool {
    !atty::is(atty::Stream::Stdout)
}

// Disable color when piped
fn should_colorize() -> bool {
    atty::is(atty::Stream::Stdout) && std::env::var("NO_COLOR").is_err()
}
```

## Progress Bars & Spinners

```rust
use indicatif::{ProgressBar, ProgressStyle, MultiProgress};
use std::time::Duration;

fn download_files(urls: &[String]) -> anyhow::Result<()> {
    let mp = MultiProgress::new();

    let overall = mp.add(ProgressBar::new(urls.len() as u64));
    overall.set_style(
        ProgressStyle::default_bar()
            .template("{spinner} [{elapsed}] {bar:40.cyan/blue} {pos}/{len} {msg}")?
            .progress_chars("█▉▊▋▌▍▎▏  "),
    );

    for url in urls {
        let pb = mp.add(ProgressBar::new(0));
        pb.set_style(
            ProgressStyle::default_bar()
                .template("  {msg} {bar:30} {bytes}/{total_bytes}")?
        );
        pb.set_message(url.clone());

        // Simulate download
        download_with_progress(url, &pb)?;
        pb.finish_and_clear();
        overall.inc(1);
    }

    overall.finish_with_message("Done!");
    Ok(())
}

// Spinner for indeterminate tasks
fn run_with_spinner<F, T>(msg: &str, f: F) -> T
where F: FnOnce() -> T
{
    let pb = ProgressBar::new_spinner();
    pb.set_style(ProgressStyle::default_spinner()
        .template("{spinner} {msg}")
        .unwrap());
    pb.set_message(msg.to_string());
    pb.enable_steady_tick(Duration::from_millis(100));
    let result = f();
    pb.finish_and_clear();
    result
}
```

## Colored Output

```rust
use colored::Colorize;

fn print_status(success: bool, message: &str) {
    if success {
        println!("{} {}", "✓".green().bold(), message);
    } else {
        eprintln!("{} {}", "✗".red().bold(), message);
    }
}

fn print_table(headers: &[&str], rows: &[Vec<String>]) {
    // header
    let header: Vec<String> = headers.iter()
        .map(|h| h.bold().underline().to_string())
        .collect();
    println!("{}", header.join("  "));

    // rows
    for row in rows {
        println!("{}", row.join("  "));
    }
}

// Respect NO_COLOR env var
fn colorize_if_supported(s: &str, color: fn(&str) -> colored::ColoredString) -> String {
    if std::env::var("NO_COLOR").is_ok() || !atty::is(atty::Stream::Stdout) {
        s.to_string()
    } else {
        color(s).to_string()
    }
}
```

## Output Formats

Support both human-readable and machine-parseable output:

```rust
#[derive(clap::ValueEnum, Clone, serde::Deserialize)]
enum OutputFormat {
    Text,
    Json,
    Table,
}

fn output_results(items: &[Item], format: &OutputFormat) -> anyhow::Result<()> {
    match format {
        OutputFormat::Json => {
            println!("{}", serde_json::to_string_pretty(items)?);
        }
        OutputFormat::Table => {
            print_table_view(items);
        }
        OutputFormat::Text => {
            for item in items {
                println!("{}: {}", item.name, item.value);
            }
        }
    }
    Ok(())
}
```

## Error Output

```rust
fn main() {
    if let Err(err) = run() {
        // Print error chain
        eprintln!("{} {}", "error:".red().bold(), err);
        for cause in err.chain().skip(1) {
            eprintln!("  {} {}", "caused by:".dimmed(), cause);
        }
        std::process::exit(1);
    }
}

// Or with better formatting:
fn main() {
    if let Err(err) = run() {
        eprintln!("{err:#}"); // anyhow's chain format
        std::process::exit(1);
    }
}
```

## Basic TUI with Ratatui

```rust
use crossterm::{
    event::{self, Event, KeyCode},
    execute,
    terminal::{disable_raw_mode, enable_raw_mode, EnterAlternateScreen, LeaveAlternateScreen},
};
use ratatui::{prelude::*, widgets::*};
use std::io;

fn run_tui() -> io::Result<()> {
    // Setup terminal
    enable_raw_mode()?;
    let mut stdout = io::stdout();
    execute!(stdout, EnterAlternateScreen)?;
    let backend = CrosstermBackend::new(stdout);
    let mut terminal = Terminal::new(backend)?;

    // App state
    let mut counter = 0u32;

    loop {
        terminal.draw(|frame| {
            let area = frame.area();
            let block = Block::new()
                .title("My TUI App")
                .borders(Borders::ALL);
            let paragraph = Paragraph::new(format!("Counter: {counter}"))
                .block(block)
                .alignment(Alignment::Center);
            frame.render_widget(paragraph, area);
        })?;

        if event::poll(std::time::Duration::from_millis(50))? {
            if let Event::Key(key) = event::read()? {
                match key.code {
                    KeyCode::Char('q') | KeyCode::Esc => break,
                    KeyCode::Up | KeyCode::Char('+') => counter += 1,
                    KeyCode::Down | KeyCode::Char('-') => counter = counter.saturating_sub(1),
                    _ => {}
                }
            }
        }
    }

    // Cleanup
    disable_raw_mode()?;
    execute!(terminal.backend_mut(), LeaveAlternateScreen)?;
    Ok(())
}
```

## Shell Completion

```rust
use clap::CommandFactory;
use clap_complete::{generate, Shell};

fn generate_completion(shell: Shell) {
    let mut cmd = Cli::command();
    let name = cmd.get_name().to_string();
    generate(shell, &mut cmd, name, &mut std::io::stdout());
}

// Add to CLI:
#[derive(Subcommand)]
enum Commands {
    // ... other commands
    /// Generate shell completion
    Completion {
        #[arg(value_enum)]
        shell: clap_complete::Shell,
    },
}
```

## Project Layout

```
my-cli/
├── src/
│   ├── main.rs       # Entry: Cli::parse() + dispatch
│   ├── cli.rs        # Clap structs
│   ├── config.rs     # Config loading/saving
│   ├── error.rs      # AppError definition
│   └── commands/
│       ├── init.rs
│       ├── run.rs
│       └── status.rs
├── tests/
│   └── integration.rs
└── Cargo.toml
```

## Release Distribution

```toml
# Use cargo-dist for GitHub release automation
[workspace.metadata.dist]
cargo-dist-version = "0.22"
ci = "github"
installers = ["shell", "powershell", "homebrew"]
targets = ["x86_64-unknown-linux-gnu", "aarch64-apple-darwin", "x86_64-pc-windows-msvc"]
```

```bash
cargo install cargo-dist
cargo dist init
cargo dist build
```

## References

- [clap docs](https://docs.rs/clap)
- [ratatui docs](https://docs.rs/ratatui) + [ratatui book](https://ratatui.rs/)
- [indicatif](https://docs.rs/indicatif)
- [cargo-dist](https://opensource.axo.dev/cargo-dist/)
- [Command Line Interface Guidelines](https://clig.dev/)
