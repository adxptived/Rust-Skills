# Arena Allocation

Arena allocation groups many allocations under one shared lifetime and frees them together.

## When to Use

Good fits: parsers, ASTs, graph building, compiler phases, and short-lived batches.

Bad fits: independent lifetimes, unbounded caches, and values needing precise destructor timing.

## Bump Allocation

```rust
use bumpalo::Bump;

let arena = Bump::new();
let name = arena.alloc_str("node");
let value = arena.alloc(42_u64);
```

## Arena-Backed AST

```rust
pub enum Expr<'a> {
    Number(i64),
    Add(&'a Expr<'a>, &'a Expr<'a>),
}

pub fn parse<'a>(arena: &'a bumpalo::Bump) -> &'a Expr<'a> {
    let left = arena.alloc(Expr::Number(1));
    let right = arena.alloc(Expr::Number(2));
    arena.alloc(Expr::Add(left, right))
}
```

## Resetting

```rust
let mut arena = Bump::new();
for batch in batches {
    process(batch, &arena);
    arena.reset();
}
```

Do not keep references from an arena after `reset`.

## Tradeoffs

Faster allocation and less fragmentation, but objects may live longer than necessary and destructors may not run individually.


## Extra Examples

Prefer examples that show both the happy path and the failure path. A good reference snippet should make ownership, errors, and lifecycle boundaries visible.

```rust
fn validate_non_empty(input: &str) -> Result<&str, &'static str> {
    if input.trim().is_empty() {
        Err("input must not be empty")
    } else {
        Ok(input)
    }
}
```

## Maintenance Notes

Keep this reference aligned with current Rust idioms. If a crate API changes, update the snippet and mention the version-sensitive part in prose.
