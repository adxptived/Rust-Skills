# CLI UX & Feedback: Terminal Interface Guidelines

Creating professional CLI tools requires clean, non-intrusive feedback loops, colored output management, progress indicator rendering, and panic management.

## 1. Colored Output & Formatting

Use the `console` crate to render colors that adapt automatically. It respects system standards (e.g. disables colors when output is redirected to a file or when `NO_COLOR` is set).

```rust
use console::style;

pub fn print_status(message: &str, is_success: bool) {
    if is_success {
        println!("{} {}", style("✔ SUCCESS:").green().bold(), message);
    } else {
        eprintln!("{} {}", style("✘ ERROR:").red().bold(), message);
    }
}
```

## 2. Progress Bars & Spinners

Use the `indicatif` crate to display terminal spinners and progress bars during long-running tasks.

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

pub fn process_items(count: u64) {
    let pb = ProgressBar::new(count);
    pb.set_style(ProgressStyle::with_template("{spinner:.green} [{elapsed_precise}] [{wide_bar:.cyan/blue}] {pos}/{len} ({eta})")
        .unwrap()
        .progress_chars("#>-"));

    for _ in 0..count {
        std::thread::sleep(Duration::from_millis(50));
        pb.inc(1);
    }
    pb.finish_with_message("Done!");
}
```

## 3. Multi-Progress Tracking

For parallel operations, use `indicatif::MultiProgress` to manage multiple progress bars simultaneously:

```rust
use indicatif::{MultiProgress, ProgressBar, ProgressStyle};

let mp = MultiProgress::new();
let styles = vec!["downloading", "compiling", "linking"];

let handles: Vec<_> = styles.iter().map(|s| {
    let pb = mp.add(ProgressBar::new(100));
    pb.set_style(ProgressStyle::with_template("{spinner} [{bar:20}] {msg} {pos}/{len}")
        .unwrap());
    pb.set_message(s);
    pb
}).collect();

// Spawn threads that update handles[i]
```

## 4. Graceful Panic Interception

User-facing CLI tools should not dump raw Rust panic messages. Use `human-panic` to capture panics and display a localized report suggestion.

```rust
use human_panic::setup_panic;

fn main() {
    setup_panic!();
    // Core logic — panics show a friendly "We're sorry" message
    // and write the actual panic details to a temp file
}
```

## 5. Structured Output Formats

```rust
#[derive(clap::Parser)]
struct Cli {
    /// Output format
    #[arg(long, value_enum, default_value = "human")]
    format: OutputFormat,
}

#[derive(clap::ValueEnum, Clone)]
enum OutputFormat { Human, Json, Yaml }

fn output_result(result: &CmdResult, format: OutputFormat) {
    match format {
        OutputFormat::Human => println!("{}", result.human_readable()),
        OutputFormat::Json => println!("{}", serde_json::to_string(result).unwrap()),
        OutputFormat::Yaml => println!("{}", serde_yaml::to_string(result).unwrap()),
    }
}
```

Provide `--json` / `--yaml` flags for machine-readable output. Detect piped stdout and switch to JSON automatically.

## 6. Error Reporting

```rust
use color_eyre::eyre::Result;
use color_eyre::install;

fn main() -> Result<()> {
    install()?; // Color-eyre spantrace + error context
    run()?;
    Ok(())
}

fn run() -> Result<()> {
    let data = std::fs::read_to_string("config.toml")
        .wrap_err("failed to read config")?;
    // ...
    Ok(())
}
```

`color-eyre` provides:
- Section spans with source locations.
- Help suggestions from error context.
- Automatic color and pager support.

## 7. Spinner for Indeterminate Tasks

```rust
use indicatif::{ProgressBar, ProgressStyle};

let spinner = ProgressBar::new_spinner();
spinner.set_style(ProgressStyle::with_template("{spinner} {msg}").unwrap());
spinner.set_message("Connecting to server...");

// Do work that has no progress percentage
connect().await;

spinner.finish_with_message("Connected!");
```

## 8. Confirmation Prompts

```rust
use dialoguer::{Confirm, Input, Select};

// Yes/no
if Confirm::new().with_prompt("Delete all data?").interact()? {
    delete_all();
}

// Text input with validation
let name: String = Input::new()
    .with_prompt("Your name")
    .validate_with(|input: &String| -> Result<(), &str> {
        if input.len() >= 3 { Ok(()) } else { Err("too short") }
    })
    .interact_text()?;

// Selection from list
let options = vec!["Small", "Medium", "Large"];
let choice = Select::new()
    .with_prompt("Pick a size")
    .items(&options)
    .interact()?;
```

## 9. Stderr vs Stdout Discipline

```rust
// Stdout: pipeline data — only output what the user asked for
println!("{}", result.id);  // OK for single-value output

// Stderr: progress, status, diagnostics — never spams pipe
eprintln!("Processed {} records", count);  // OK — goes to stderr
eprintln!("{} {}", style("WARN:").yellow(), "deprecated field");
```

Rule: if `--quiet` or piped stdout should suppress it, it goes to stderr.

## 10. TTY Detection

```rust
use atty::Stream;

fn welcome_message() {
    if atty::is(Stream::Stdout) {
        println!("Welcome to mycli v{}", env!("CARGO_PKG_VERSION"));
    }
    // If piped, don't print banner
}

fn spinner_if_tty() {
    if atty::is(Stream::Stderr) {
        // Show progress bars on stderr TTY
    }
}
```

## 11. Anti-Patterns

- Mixing progress output and data output on stdout.
- Using `unwrap()` in CLI tools — always use proper error messages.
- Writing colors manually via ANSI codes instead of using `console`/`colored`.
- Blocking the event loop with an unresponsive spinner.
- Displaying a spinner for sub-second operations.
