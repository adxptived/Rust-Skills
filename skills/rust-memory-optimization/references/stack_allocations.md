# Stack-Friendly Allocations

Reduce heap traffic in hot paths by keeping small data inline.

## SmallVec

Use when most collections are small but occasionally grow.

```rust
use smallvec::SmallVec;

type Tags = SmallVec<[Tag; 4]>;
let mut tags = Tags::new();
tags.push(Tag::FastPath);
```

## ArrayVec

Use when capacity is strictly bounded.

```rust
use arrayvec::ArrayVec;

let mut buf: ArrayVec<u8, 64> = ArrayVec::new();
buf.try_extend_from_slice(b"hello")?;
```

## Compact Strings

Use `compact_str` or `smol_str` for many short strings.

## Boxed Slices

Use `Box<[T]>` after construction to drop unused vector capacity.

```rust
let mut values = Vec::with_capacity(1024);
load_values(&mut values);
let values: Box<[Value]> = values.into_boxed_slice();
```

## Measure First

Inline storage increases stack size and move cost. Benchmark before replacing every `Vec`.

```rust
assert!(std::mem::size_of::<SmallVec<[u8; 32]>>() > std::mem::size_of::<Vec<u8>>());
```


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
