# Soundness Checklist

Unsafe code is acceptable only when a safe abstraction prevents undefined behavior for all safe callers.

## Safety Comments

Every unsafe block needs a local invariant explanation.

```rust
// SAFETY: `ptr` was created from `slice.as_ptr()`, is aligned, and `idx < slice.len()`.
unsafe { *ptr.add(idx) }
```

## Unsafe Functions

Mark functions `unsafe` when callers must uphold invariants the function cannot verify.

```rust
/// # Safety
/// `ptr` must be non-null, aligned, and valid for reads of `len` bytes.
pub unsafe fn bytes_from_raw<'a>(ptr: *const u8, len: usize) -> &'a [u8] {
    unsafe { std::slice::from_raw_parts(ptr, len) }
}
```

## Invariants

Write down pointer validity, alignment, initialization, aliasing, lifetime relationship, thread-safety assumptions, and deallocation responsibility.

## Safe Wrapper Rule

A safe public API must not allow UB for any input.

## FFI

Use `repr(C)`, explicit ownership transfer docs, null checks, and panic boundaries.

## Review Questions

Can safe callers violate invariants? Are lifetimes tied to allocation? Do destructors run exactly once? Does Miri cover representative paths?


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
