# Rust Skills for AI Agents

A comprehensive collection of Rust programming skills for AI coding assistants (Claude, Devin, Cursor, OpenCode, etc.).

These skills provide AI agents with deep knowledge of Rust idioms, patterns, and best practices — from ownership fundamentals to production web services and embedded firmware.

## Skills

### Core Language

| Skill | Description |
|-------|-------------|
| **rust-mastery** | Core Rust: ownership, borrowing, lifetimes, traits, iterators — the mental model |
| **rust-ownership** | Deep dive: borrow checker errors, lifetimes, Cow, interior mutability |
| **rust-type-system** | Type-driven design: newtype, typestate, PhantomData, sealed traits |
| **rust-error-handling** | Result/Option, thiserror, anyhow, custom errors, error context |
| **rust-patterns** | Builder, Newtype, State Machine, RAII, Typestate — idiomatic patterns |
| **rust-api-design** | Public API design: naming, trait ergonomics, attributes, semver-safe APIs |
| **rust-unsafe** | Unsafe Rust: soundness invariants, FFI, raw pointers, Miri validation |

### Systems & Performance

| Skill | Description |
|-------|-------------|
| **rust-performance** | Profiling, benchmarking, allocations, Cow, SmallVec, compiler optimization |
| **rust-memory-optimization** | Allocation reduction, stack vs heap, arenas, memory layout |
| **rust-concurrency** | OS threads, Arc/Mutex, channels, rayon, atomics, deadlock debugging |
| **rust-async** | Tokio, futures, channels, JoinSet, CancellationToken, graceful shutdown |
| **rust-embedded** | no_std, Embassy, RTIC, embedded-hal, MCU targets, defmt logging |

### Domain & Frameworks

| Skill | Description |
|-------|-------------|
| **rust-axum** | Axum HTTP API: routing, extractors, state, middleware, WebSockets |
| **rust-database** | Database apps: SQLx, Diesel, SeaORM, migrations, pools, transactions |
| **rust-dioxus** | Dioxus UI: signals, components, routing, server functions (web/desktop) |
| **rust-wasm** | WebAssembly: wasm-bindgen, wasm-pack, web-sys, JS interop, bundle size |
| **rust-cli** | CLI tools: clap, TUI with ratatui, progress bars, config files |
| **rust-cloud-native** | Containers, Kubernetes, observability, graceful shutdown, deployments |
| **rust-fintech** | Decimal arithmetic, ledgers, idempotency, audit-safe transaction systems |
| **rust-ml** | DataFrames, tensors, candle/tch/polars, model serving and pipelines |
| **rust-networking** | TCP/UDP sockets, TLS, WebSockets, HTTP clients, gRPC, custom protocols |
| **rust-observability** | Tracing, metrics, OpenTelemetry, structured logging, production diagnostics |
| **rust-security** | AppSec: secrets, password hashing, supply-chain checks, threat modeling |

### Ecosystem & Craft

| Skill | Description |
|-------|-------------|
| **rust-ecosystem** | Crate selection, workspaces, feature flags, Cargo plugins, CI |
| **rust-testing** | Unit/integration tests, proptest, mockall, criterion, insta snapshots |

## Installation

### Quick Install (npx)

```bash
npx skills add adxptived/Rust-Skills
```

### Git Clone

```bash
# Global (all projects)
git clone https://github.com/adxptived/Rust-Skills.git ~/.agents/skills/rust-skills

# Local (current project only)
git clone https://github.com/adxptived/Rust-Skills.git .agents/skills/rust-skills

# For OpenCode
git clone https://github.com/adxptived/Rust-Skills.git .opencode/skills/rust-skills

# For Claude
git clone https://github.com/adxptived/Rust-Skills.git .claude/skills/rust-skills
```

### Specific Skills Only

```bash
git clone --filter=blob:none --sparse https://github.com/adxptived/Rust-Skills.git
cd Rust-Skills
git sparse-checkout set skills/rust-axum skills/rust-async skills/rust-error-handling
cp -r skills/* ~/.agents/skills/
```

## Usage

### Automatic Triggering

Skills trigger automatically based on context. Just ask naturally:

```
"Why am I getting 'cannot borrow as mutable' error?"
→ rust-ownership, rust-mastery

"How do I limit concurrent requests in Tokio?"
→ rust-async

"My Rust code is slow, how do I profile it?"
→ rust-performance

"Should I use thiserror or anyhow?"
→ rust-error-handling

"How do I build a REST API in Rust?"
→ rust-axum

"How should I structure SQLx migrations and transactions?"
→ rust-database

"How do I add structured logging and metrics to my Axum service?"
→ rust-observability

"How do I store passwords and audit Rust dependencies safely?"
→ rust-security

"How do I make invalid states unrepresentable?"
→ rust-type-system

"I need a Dioxus component with async data fetching"
→ rust-dioxus

"How do I expose Rust image processing to a browser app?"
→ rust-wasm

"How do I blink an LED on STM32 with Embassy?"
→ rust-embedded

"How do I add subcommands to my CLI tool?"
→ rust-cli
```

### Manual Invocation

```
"Using rust-axum skill, show me authentication middleware"
"/skill rust-testing"
"@rust-concurrency explain deadlock prevention"
```

## Structure

```
skills/
├── rust-mastery/              SKILL.md  — core Rust concepts + references/
├── rust-ownership/            SKILL.md  — borrow checker, lifetimes, Cow
├── rust-type-system/          SKILL.md  — newtype, typestate, PhantomData
├── rust-error-handling/       SKILL.md  — Result, Option, thiserror, anyhow
├── rust-patterns/             SKILL.md  — design patterns + references/
├── rust-api-design/           SKILL.md  — public API guidelines + references/
├── rust-unsafe/               SKILL.md  — unsafe invariants, FFI, Miri
├── rust-performance/          SKILL.md  — profiling, optimization + references/
├── rust-memory-optimization/  SKILL.md  — allocations, arenas, stack/heap
├── rust-concurrency/          SKILL.md  — threads, channels, rayon, atomics
├── rust-async/                SKILL.md  — Tokio, futures, JoinSet + references/
├── rust-embedded/             SKILL.md  — no_std, Embassy, RTIC, MCUs
├── rust-axum/                 SKILL.md  — Axum web framework
├── rust-database/             SKILL.md  — SQLx, migrations, pools, transactions
├── rust-dioxus/               SKILL.md  — Dioxus UI framework
├── rust-wasm/                 SKILL.md  — WebAssembly bindings and bundling
├── rust-cli/                  SKILL.md  — CLI tools, ratatui TUI
├── rust-cloud-native/         SKILL.md  — containers, observability, deploys
├── rust-fintech/              SKILL.md  — decimals, ledgers, transactions
├── rust-ml/                   SKILL.md  — Polars, tensors, model serving
├── rust-networking/           SKILL.md  — TCP/UDP, TLS, WebSockets, gRPC
├── rust-security/             SKILL.md  — AppSec, secrets, supply-chain hardening
├── rust-observability/        SKILL.md  — tracing, metrics, OpenTelemetry + references/
├── rust-ecosystem/            SKILL.md  — crates, Cargo, CI, publishing
└── rust-testing/              SKILL.md  — tests, proptest, mockall, criterion
```

## Sources

**Books:**
- The Rust Programming Language (Klabnik & Nichols)
- Programming Rust, 2nd Edition (Blandy & Orendorff)
- Effective Rust (Drysdale)
- Rust Atomics and Locks (Mara Bos)

**Official Documentation:**
- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust Reference](https://doc.rust-lang.org/reference/)
- [Async Book](https://rust-lang.github.io/async-book/)
- [Rustonomicon](https://doc.rust-lang.org/nomicon/)
- [The Embedded Rust Book](https://docs.rust-embedded.org/book/)

**Community Resources:**
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/)
- [Rust Performance Book](https://nnethercote.github.io/perf-book/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [blessed.rs](https://blessed.rs) — curated crate recommendations

**Reference Implementations:**
- [tokio](https://github.com/tokio-rs/tokio), [axum](https://github.com/tokio-rs/axum)
- [ripgrep](https://github.com/BurntSushi/ripgrep), [fd](https://github.com/sharkdp/fd)
- [dioxus](https://github.com/DioxusLabs/dioxus)
- [embassy](https://github.com/embassy-rs/embassy)

## Contributing

1. Follow the existing SKILL.md format (frontmatter with name/description triggers)
2. Include practical, copy-paste-ready code examples
3. Include anti-patterns alongside correct patterns
4. Cite sources in References section
5. Test with an AI agent before submitting

## License

MIT License - see [LICENSE](LICENSE)
