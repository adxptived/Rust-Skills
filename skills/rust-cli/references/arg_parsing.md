# CLI Argument Parsing: Advanced Clap Integration

Clap (Command Line Argument Parser) v4 is the industry standard for building powerful command-line interfaces in Rust.

## 1. Derive API Structure

The Derive API is the recommended way to parse arguments using declarative Rust structs.

```rust
use clap::{Parser, Subcommand, Args};

#[derive(Parser, Debug)]
#[command(name = "myapp")]
#[command(author = "Your Name <email@example.com>")]
#[command(version = "1.0")]
#[command(about = "Does awesome things", long_about = None)]
pub struct Cli {
    /// Turn debugging information on
    #[arg(short, long, action = clap::ArgAction::Count)]
    pub debug: u8,

    /// Optional custom config path
    #[arg(short, long, value_name = "FILE")]
    pub config: Option<std::path::PathBuf>,

    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand, Debug)]
pub enum Commands {
    /// Import data into the application
    Import(ImportArgs),
    /// Export logs to a destination
    Export {
        #[arg(short, long)]
        output: String,
    },
}

#[derive(Args, Debug)]
pub struct ImportArgs {
    /// Path to CSV file containing values
    pub path: std::path::PathBuf,

    /// Skip validation of values
    #[arg(long)]
    pub skip_validation: bool,
}

fn main() {
    let args = Cli::parse();

    match args.command {
        Commands::Import(import_args) => {
            println!("Importing from {:?}", import_args.path);
        }
        Commands::Export { output } => {
            println!("Exporting to {}", output);
        }
    }
}
```

## 2. Advanced Validation & Default Values

Clap allows specifying environment variables as fallbacks and implementing custom parser validators.

```rust
#[derive(Parser)]
pub struct AppConfig {
    /// Port of the server, fallbacks to $PORT env, default is 8080
    #[arg(
        short,
        long,
        env = "PORT",
        default_value = "8080",
        value_parser = clap::value_parser!(u16)
    )]
    pub port: u16,

    /// Domain name validation
    #[arg(short, long, value_parser = validate_domain)]
    pub domain: String,
}

fn validate_domain(val: &str) -> Result<String, String> {
    if val.contains('.') && val.len() > 3 {
        Ok(val.to_string())
    } else {
        Err(String::from("domain must be a valid dotted string (e.g. example.com)"))
    }
}
```

### Custom Value Parsers

```rust
use std::net::SocketAddr;
use std::str::FromStr;

/// Parse a comma-separated list of socket addresses
fn parse_addr_list(s: &str) -> Result<Vec<SocketAddr>, String> {
    s.split(',')
        .map(|addr| SocketAddr::from_str(addr.trim()).map_err(|e| e.to_string()))
        .collect()
}

#[derive(Parser)]
struct ServerConfig {
    #[arg(short, long, value_parser = parse_addr_list, default_value = "127.0.0.1:8080")]
    addrs: Vec<SocketAddr>,
}
```

### Default Values from Functions

```rust
fn default_log_dir() -> PathBuf {
    dirs::data_dir()
        .unwrap_or_else(|| PathBuf::from("."))
        .join("myapp")
        .join("logs")
}

fn default_threads() -> usize {
    std::thread::available_parallelism()
        .map(|n| n.get())
        .unwrap_or(4)
}

#[derive(Parser)]
struct AppConfig {
    #[arg(long, default_value_t = default_log_dir())]
    log_dir: PathBuf,

    #[arg(long, default_value_t = default_threads())]
    threads: usize,
}
```

## 3. Subcommand Patterns

### Flatten: Shared Subcommand State

```rust
#[derive(Args)]
struct CommonArgs {
    /// API endpoint
    #[arg(short, long, global = true, default_value = "https://api.example.com")]
    endpoint: String,

    /// API key
    #[arg(short, long, global = true, env = "API_KEY")]
    api_key: Option<String>,
}

#[derive(Parser)]
struct Cli {
    #[command(flatten)]
    common: CommonArgs,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    List,
    Create {
        #[arg(short, long)]
        name: String,
        #[command(flatten)]
        common: CommonArgs,  // re-flatten - clap merges values
    },
}
```

### Subcommand with Shared Mutable State

```rust
// Each subcommand can carry the same flattened struct;
// clap merges the values when both global and subcommand specify a field.
```

## 4. Arg Groups and Requirements

```rust
#[derive(Parser)]
struct Config {
    // Either --input-file OR --input-url, not both
    #[arg(short = 'f', long, group = "input")]
    input_file: Option<PathBuf>,

    #[arg(short = 'u', long, group = "input")]
    input_url: Option<String>,

    // --output and --dry-run conflict
    #[arg(short, long, conflicts_with = "dry_run")]
    output: Option<PathBuf>,

    #[arg(long, conflicts_with = "output")]
    dry_run: bool,

    // Requires --user when --auth is present
    #[arg(long, requires = "user")]
    auth: Option<String>,

    #[arg(long)]
    user: Option<String>,
}

// Define explicit groups
#[derive(Parser)]
#[command(group = clap::ArgGroup::new("action")
    .args(["create", "delete", "update"])
    .required(true)
    .multiple(false))]
struct Cli {
    #[arg(long)]
    create: bool,
    #[arg(long)]
    delete: bool,
    #[arg(long)]
    update: bool,
}
```

## 5. Action Types

| Action | Behavior | Use case |
|--------|----------|----------|
| `Set` | Overwrites any previous value | Standard flags |
| `Append` | Collects multiple occurrences in a `Vec` | `-v -v -v` |
| `Count` | Counts occurrences as `u8` | Verbosity levels |
| `SetTrue` / `SetFalse` | Bool flag, auto-generated | Enable/disable toggles |
| `Version` | Prints version and exits | `--version` |
| `Help` | Prints help and exits | `--help` / `-h` |
| `HelpShort` / `HelpLong` | Help with specific style | Custom help behavior |

### Count Pattern for Verbosity

```rust
#[derive(Parser)]
struct Cli {
    /// Verbosity level (-v, -vv, -vvv)
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,
}

fn init_logging(verbose: u8) {
    let level = match verbose {
        0 => tracing::Level::ERROR,
        1 => tracing::Level::WARN,
        2 => tracing::Level::INFO,
        3 => tracing::Level::DEBUG,
        _ => tracing::Level::TRACE,
    };
    tracing_subscriber::fmt().with_max_level(level).init();
}
```

## 6. Shell Completion Generation

```rust
use clap::CommandFactory;
use clap_complete::{generate, Generator, Shell};

#[derive(Subcommand)]
enum Commands {
    /// Generate shell completion
    Completion {
        #[arg(value_enum)]
        shell: Shell,
    },
}

fn print_completions<G: Generator>(gen: G, cmd: &mut clap::Command) {
    generate(gen, cmd, cmd.get_name().to_string(), &mut std::io::stdout());
}

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Commands::Completion { shell } => {
            let mut cmd = Cli::command();
            print_completions(shell, &mut cmd);
        }
        _ => { /* normal flow */ }
    }
}
```

Install completions:
```bash
# Bash
myapp completion bash > /etc/bash_completion.d/myapp

# Zsh
myapp completion zsh > /usr/local/share/zsh/site-functions/_myapp

# Fish
myapp completion fish > ~/.config/fish/completions/myapp.fish

# PowerShell
myapp completion powershell >> $PROFILE
```

## 7. Color and Help Customization

```rust
#[derive(Parser)]
#[command(
    name = "myapp",
    version,
    about,
    // Custom help template
    help_template = "\
{before-help}{name} v{version}
{about-with-newline}
{usage-heading} {usage}

{all-args}{after-help}
",
    // Disable auto-generated -h short flag
    disable_help_flag = true,
)]
struct Cli {
    #[arg(long, action = clap::ArgAction::Help)]
    help: Option<bool>,
}
```

### Override Color Auto-Detection

```rust
#[derive(Parser)]
#[command(color = clap::ColorChoice::Auto)]
struct Cli;

// clap respects NO_COLOR, CLICOLOR, and CLICOLOR_FORCE
```

## 8. Conflict/Exclusion Rules Reference

```rust
#[arg(long, conflicts_with = "daemon")]
foreground: bool,

#[arg(long, requires = "database_url")]
run_migrations: bool,

#[arg(long, requires_all = &["host", "port"])]
connect: bool,

// Mutual exclusion shorthand
#[arg(long, group = "mode")]
serve: bool,
#[arg(long, group = "mode")]
migrate: bool,
```

## 9. Builder API (When Derive Isn't Enough)

```rust
use clap::{Arg, Command};

fn build_cli() -> Command {
    Command::new("myapp")
        .version("1.0")
        .about("Complex CLI")
        .arg(
            Arg::new("config")
                .short('c')
                .long("config")
                .value_name("FILE")
                .default_value("config.toml")
                .env("CONFIG_PATH"),
        )
        .subcommand(
            Command::new("run")
                .about("Run the service")
                .arg(
                    Arg::new("port")
                        .short('p')
                        .default_value("8080")
                        .value_parser(clap::value_parser!(u16)),
                ),
        )
}
```

Use the Builder API when you need dynamic subcommands, generated arguments, or when the struct shape doesn't map cleanly to derive.

## 10. Anti-Patterns

```rust
// Bad: unwrap on parse — clap errors are user-facing, not programmer errors
let args = Cli::parse(); // clap handles --help and errors internally

// Bad: using unwrap in custom validators
fn validate_port(val: &str) -> Result<String, String> {
    let n: u16 = val.parse().unwrap(); // panics! Return Err instead
    Ok(val.to_string())
}

// Good: return Err for validation failures
fn validate_port(val: &str) -> Result<String, String> {
    val.parse::<u16>().map_err(|_| "invalid port number".to_string())?;
    Ok(val.to_string())
}

// Bad: global flags defined on every subcommand instead of using global = true
// Bad: too many positional arguments (hard to remember order — use named flags)
// Bad: hiding errors behind 'quiet' with no way to see the error
```
