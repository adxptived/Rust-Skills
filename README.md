# Rust Skills for AI Agents

A comprehensive collection of Rust programming skills for AI coding assistants (Claude, Devin, Cursor, etc.).

These skills provide AI agents with deep knowledge of Rust idioms, patterns, and best practices — enabling them to write better Rust code and provide more accurate guidance.

## Skills

| Skill | Description |
|-------|-------------|
| **rust-mastery** | Core Rust concepts: ownership, borrowing, lifetimes, traits, error handling |
| **rust-performance** | Profiling, benchmarking, optimization techniques, reducing allocations |
| **rust-async** | Async/await, Tokio runtime, futures, channels, concurrent patterns |
| **rust-error-handling** | Result/Option patterns, thiserror, anyhow, custom error types |
| **rust-patterns** | Design patterns: Builder, Newtype, State Machine, RAII, Typestate |

## Installation

### Quick Install (npx)

```bash
npx add-skill adxptived/Rust-Skills
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

### Manual Download

```bash
# Download and extract
curl -L https://github.com/adxptived/Rust-Skills/archive/main.tar.gz | tar -xz
cp -r Rust-Skills-main/skills/* ~/.agents/skills/
```

### Specific Skills Only

```bash
# Clone with sparse checkout
git clone --filter=blob:none --sparse https://github.com/adxptived/Rust-Skills.git
cd Rust-Skills
git sparse-checkout set skills/rust-mastery skills/rust-async
cp -r skills/* ~/.agents/skills/
```

## Usage

### Automatic Triggering

Skills trigger automatically based on context. Just ask questions naturally:

```
"Why am I getting 'cannot borrow as mutable' error?"
→ triggers rust-mastery

"How do I limit concurrent requests in Tokio?"
→ triggers rust-async

"My Rust code is slow, how do I profile it?"
→ triggers rust-performance

"Should I use thiserror or anyhow?"
→ triggers rust-error-handling

"How do I implement a builder pattern?"
→ triggers rust-patterns
```

### Manual Invocation

Explicitly request a skill:

```
"Using rust-mastery skill, explain lifetimes"
"@rust-async help me with channels"
"/skill rust-performance"
```

### Working with References

Skills have deep-dive reference files. Ask to load them:

```
"Read the ownership reference and explain borrowing"
"Check references/tokio.md for runtime configuration"
```

### Example Session

```
User: I have this Rust code with a borrow error:
      let mut v = vec![1,2,3];
      let first = &v[0];
      v.push(4);
      println!("{}", first);

Agent: [rust-mastery skill activates]
       This violates Rust's borrowing rules...
       [provides fix with explanation]

User: Now make it faster for large vectors

Agent: [rust-performance skill activates]
       Use Vec::with_capacity() to pre-allocate...
       [provides optimized code]
```

## Structure

```
skills/
├── rust-mastery/
│   ├── SKILL.md              # Core Rust concepts
│   └── references/
│       ├── ownership.md      # Ownership, borrowing, lifetimes
│       ├── traits.md         # Traits, generics, trait objects
│       ├── error-handling.md # Result, Option patterns
│       └── idioms.md         # Common idioms and anti-patterns
│
├── rust-performance/
│   ├── SKILL.md              # Optimization overview
│   └── references/
│       ├── profiling.md      # CPU/memory profiling tools
│       ├── allocations.md    # Reducing heap allocations
│       └── concurrency.md    # Parallel performance
│
├── rust-async/
│   ├── SKILL.md              # Async programming overview
│   └── references/
│       ├── tokio.md          # Tokio runtime deep dive
│       ├── patterns.md       # Async patterns & anti-patterns
│       └── channels.md       # Message passing in depth
│
├── rust-error-handling/
│   └── SKILL.md              # thiserror, anyhow, error design
│
└── rust-patterns/
    ├── SKILL.md              # Design patterns overview
    └── references/
        ├── creational.md     # Builder, Factory, Singleton
        ├── structural.md     # Newtype, Wrapper, Extension
        └── behavioral.md     # State Machine, Strategy, Command
```

## What's Covered

### rust-mastery
- Ownership and borrowing rules
- Lifetime annotations and elision
- Trait system and generics
- Standard traits (Debug, Clone, Default, From, etc.)
- Iterator patterns
- Common compilation errors and fixes

### rust-performance
- Profiling with perf, flamegraph, samply
- Benchmarking with Criterion
- Memory profiling with DHAT, heaptrack
- Reducing allocations (Cow, pre-allocation, avoiding clones)
- Data structure selection
- Compiler optimization hints

### rust-async
- Tokio runtime configuration
- Task spawning (spawn, spawn_blocking, spawn_local)
- Concurrency primitives (join!, select!, FuturesUnordered)
- Channels (mpsc, oneshot, broadcast, watch)
- Synchronization (Mutex, RwLock, Semaphore)
- Async I/O patterns
- Graceful shutdown, rate limiting, timeouts

### rust-error-handling
- Result and Option combinators
- The ? operator
- Custom error types
- thiserror for libraries
- anyhow for applications
- Error context and chaining
- When to panic vs return Result

### rust-patterns
- Builder pattern (standard and typestate)
- Newtype pattern for type safety
- Extension traits
- RAII guards
- State machines with enums
- Typestate pattern
- Strategy and Command patterns
- Common anti-patterns to avoid

## Sources

These skills synthesize knowledge from:

**Books:**
- The Rust Programming Language (Steve Klabnik, Carol Nichols)
- Programming Rust, 2nd Edition (Jim Blandy, Jason Orendorff)
- Effective Rust (David Drysdale)
- Rust in Action (Tim McNamara)

**Official Documentation:**
- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust Reference](https://doc.rust-lang.org/reference/)
- [Async Book](https://rust-lang.github.io/async-book/)
- [Rustonomicon](https://doc.rust-lang.org/nomicon/)

**Community Resources:**
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/)
- [Rust Performance Book](https://nnethercote.github.io/perf-book/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial)

**Reference Codebases:**
- [ripgrep](https://github.com/BurntSushi/ripgrep)
- [tokio](https://github.com/tokio-rs/tokio)
- [serde](https://github.com/serde-rs/serde)
- [anyhow](https://github.com/dtolnay/anyhow) / [thiserror](https://github.com/dtolnay/thiserror)

## Skill Format

Each skill follows the standard SKILL.md format:

```markdown
---
name: skill-name
description: When to trigger this skill and what it does.
---

# Skill Title

Content with examples, patterns, and guidance.
```

Skills can include `references/` subdirectories for deep-dive documentation that the agent loads on demand.

## Contributing

Contributions welcome! Please:

1. Follow existing skill structure
2. Include practical code examples
3. Cite sources where applicable
4. Test with an AI agent before submitting

## License

MIT License - see [LICENSE](LICENSE)
