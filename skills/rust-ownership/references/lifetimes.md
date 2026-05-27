# Lifetime Patterns

Lifetimes describe how references relate. They do not extend how long values live.

## Elision

Many signatures do not need explicit lifetimes.

```rust
fn first_word(input: &str) -> &str { input.split_whitespace().next().unwrap_or("") }
```

The output lifetime is tied to `input`. Rust's lifetime elision rules apply:
1. Each input reference gets its own lifetime.
2. If there is exactly one input lifetime, it is assigned to all output lifetimes.
3. If there are multiple input lifetimes but one is `&self` or `&mut self`, the output lifetime is `&self`'s lifetime.

## Multiple Inputs

Add a lifetime when the return value may come from either input.

```rust
fn longer<'a>(left: &'a str, right: &'a str) -> &'a str {
    if left.len() >= right.len() { left } else { right }
}
```

Here both inputs must live at least as long as the returned reference.

## Structs with References

Reference-holding structs must carry lifetimes.

```rust
pub struct Parser<'src> {
    source: &'src str,
}

impl<'src> Parser<'src> {
    pub fn new(source: &'src str) -> Self { Self { source } }
}
```

### Ownership vs Borrow Trade-off

| Approach | Pros | Cons |
|----------|------|------|
| Owned (`String`) | No lifetime, no borrow checker friction | Allocation on struct creation |
| `Cow<'a, str>` | Borrow when possible, own when needed | Slightly more complex API |
| `&'a str` | Zero-copy, no allocation | Lifetime pervasive, struct scope limited |
| `Arc<str>` | Shared ownership, thread-safe | Overhead for single-owner case |

## Higher-Ranked Trait Bounds (HRTB)

Use HRTB when a function must accept references of any lifetime.

```rust
fn with_str<F>(f: F)
where
    F: for<'a> Fn(&'a str),
{
    f("value");
}
```

Without `for<'a>`, the bound would mean a specific lifetime, not all lifetimes — the function could not pass arbitrary strings.

### Common HRTB Patterns

```rust
// Closure that accepts any reference
fn process<F>(f: F) where F: Fn(&[u8]) -> bool;

// Trait bound for any lifetime
trait Processor: for<'a> Fn(&'a str) -> String;

// Iterator of references
fn first<'a, I>(iter: I) -> Option<&'a str>
where I: Iterator<Item = &'a str>;
```

## `'static`

`'static` means the reference can live for the entire program, or the owned type contains no non-static references.

```rust
fn spawn_task(msg: String) {
    tokio::spawn(async move {
        println!("{msg}");
    });
}
```

The task is `'static` because it owns `msg`, not because `msg` leaks forever.

### When You Actually Need `'static`

- Thread/task spawning (`thread::spawn`, `tokio::spawn`)
- Storing in a `static` or `const`
- Trait objects stored without lifetime (`Box<dyn Trait>`)
- Callbacks that must outlive the caller

### When You Don't (but think you might)

```rust
// Needlessly restrictive
fn process(data: &'static str) -> &'static str { data }

// Better — let the caller decide
fn process<'a>(data: &'a str) -> &'a str { data }
```

## Lifetime in Associated Types

```rust
trait Repository {
    type Item<'a> where Self: 'a;
    fn get<'a>(&'a self, id: &str) -> Self::Item<'a>;
}
```

Generic Associated Types (GATs) let traits produce references tied to the borrow.

## NLL (Non-Lexical Lifetimes)

Since Rust 2018, the borrow checker uses NLL: borrows end when they are last used, not at the end of the scope.

```rust
let mut data = vec![1, 2, 3];
let first = &data[0];       // borrow starts
println!("{first}");        // borrow ends here — last use
data.push(4);               // OK: mutable borrow not conflicting
```

Understanding NLL reduces the urge to add explicit drops or scopes.

## Lifetime Subtyping: Variance

| Variance | `&'a T` | `&'a mut T` | `Box<T>` | `fn(T) -> U` |
|----------|---------|-------------|----------|-------------|
| Covariant in | `'a`, `T` | `'a` | `T` | `U` |
| Invariant in | — | `T` | — | `T` |
| Contravariant in | — | — | — | — |

In practice: `&mut T` is invariant in `T` — you cannot coerce `&mut &'short str` to `&mut &'long str`.

## Avoid Lifetime Overuse

If every function has the same explicit lifetime, the design may be over-borrowed. Consider:
- Owned values (`String` instead of `&str`)
- `Cow<'a, T>` for write-or-borrow decisions
- `Arc<T>` for shared ownership across threads
- Smaller scoped borrows instead of struct-level lifetimes
