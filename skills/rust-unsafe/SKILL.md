---
name: rust-unsafe
description: |
  Rules and checks for writing safe Unsafe Rust. Use when:
  - User writes 'unsafe' blocks, functions, or traits
  - Dereferencing raw pointers (*const T, *mut T)
  - Interacting with FFI (Foreign Function Interfaces)
  - Performing casting or reinterpretations (transmute, union access)
  - Debugging undefined behavior or running Miri verification tests
  - Implementing custom allocators, Vec-like collections, or lock-free data structures
  - Using Pin, ManuallyDrop, MaybeUninit, or NonNull for low-level control
---

## Quick Navigation

- [references/soundness.md](references/soundness.md)
- [references/miri.md](references/miri.md)

# Unsafe Rust Programming Guide

Guidelines for maintaining soundness, preventing Undefined Behavior (UB), and proving safety boundaries.

## Core Safety Constraints

### 1. Mandatory Safety Explanations
Every single `unsafe` block or function MUST have a preceding `// SAFETY:` comment explaining exactly why it is safe and how invariants are preserved.

```rust
let my_vec = vec![1, 2, 3];
let ptr = my_vec.as_ptr();

// SAFETY: `ptr` is guaranteed to be valid and aligned for reading
// up to length `my_vec.len()`, because it points to the vec's heap data.
unsafe {
    let first = *ptr;
    assert_eq!(first, 1);
}
```

### 2. Soundness Boundaries
Unsafe blocks must build a safe abstraction layer. A safe public function must not trigger Undefined Behavior for *any* possible inputs.

```rust
// Bad: Calling this with an out-of-bounds offset is undefined behavior,
// but the function is marked as safe!
pub fn get_value_unchecked(slice: &[i32], offset: usize) -> i32 {
    unsafe { *slice.as_ptr().add(offset) }
}

// Good: Keep the function private or mark the function itself as unsafe,
// requiring callers to guarantee the index validity.
pub unsafe fn get_value_unchecked_safe(slice: &[i32], offset: usize) -> i32 {
    // SAFETY: Caller must guarantee that `offset` is within bounds
    unsafe { *slice.as_ptr().add(offset) }
}
```

### 3. Strict Pointer Aliasing Rules
Never violate Rust's aliasing rules (one mutable reference XOR multiple immutable references) even when using raw pointers.
- Converting raw pointers directly to `&mut T` while other references exist is Undefined Behavior.
- Avoid using `std::mem::transmute` if you can convert pointers or references via `as`.

```rust
let mut val = 42;
let r1 = &mut val as *mut i32;

// SAFETY: Exclusive borrow is maintained; no other references exist to `val`
// during the lifetime of the returned mutable reference.
let ref_mut = unsafe { &mut *r1 };
*ref_mut = 100;
```

### 4. FFI Safety
When exposing or calling external C functions, always handle null pointers, ensure string slices are null-terminated, and maintain correct layout representations using `#[repr(C)]`.

```rust
extern "C" {
    fn c_process_data(data: *const u8, len: libc::size_t);
}

pub fn process_data(data: &[u8]) {
    // SAFETY: The external function only reads from the pointer for `len` bytes,
    // and we pass a valid pointer and matching length from our slice.
    unsafe {
        c_process_data(data.as_ptr(), data.len() as libc::size_t);
    }
}
```

## MaybeUninit Patterns

Use `MaybeUninit` for uninitialized memory instead of `mem::zeroed()`:

```rust
use std::mem::MaybeUninit;

pub struct RingBuffer<T> {
    buffer: Box<[MaybeUninit<T>]>,
    head: usize,
    tail: usize,
    capacity: usize,
}

impl<T> RingBuffer<T> {
    pub fn with_capacity(cap: usize) -> Self {
        let mut buffer = Vec::with_capacity(cap);
        // SAFETY: We only access initialized elements; rest are MaybeUninit.
        unsafe { buffer.set_len(cap); }
        Self {
            buffer: buffer.into_boxed_slice(),
            head: 0,
            tail: 0,
            capacity: cap,
        }
    }

    pub fn push(&mut self, value: T) -> Result<(), T> {
        if self.len() == self.capacity {
            return Err(value); // buffer full
        }
        self.buffer[self.tail] = MaybeUninit::new(value);
        self.tail = (self.tail + 1) % self.capacity;
        Ok(())
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.head == self.tail {
            return None; // empty
        }
        // SAFETY: head always points to an initialized element when non-empty.
        let value = unsafe { self.buffer[self.head].assume_init_read() };
        self.head = (self.head + 1) % self.capacity;
        Some(value)
    }
}

impl<T> Drop for RingBuffer<T> {
    fn drop(&mut self) {
        // Drop only the initialized elements
        while let Some(_) = self.pop() {}
    }
}
```

## Custom Vec Implementation

A minimal unsafe-backed Vec showing key safety invariants:

```rust
use std::alloc::{alloc, dealloc, handle_alloc_error, Layout};
use std::ptr::NonNull;

pub struct RawVec<T> {
    ptr: NonNull<T>,
    cap: usize,
}

impl<T> RawVec<T> {
    pub fn with_capacity(cap: usize) -> Self {
        if cap == 0 {
            return Self { ptr: NonNull::dangling(), cap: 0 };
        }
        let layout = Layout::array::<T>(cap).unwrap();
        // SAFETY: layout is guaranteed to be non-zero by the cap check.
        let ptr = unsafe { alloc(layout) as *mut T };
        let ptr = NonNull::new(ptr).unwrap_or_else(|| handle_alloc_error(layout));
        Self { ptr, cap }
    }

    /// # Safety
    /// Caller must ensure the memory is valid for T at the returned pointer.
    pub unsafe fn ptr(&self) -> *mut T { self.ptr.as_ptr() }

    pub fn cap(&self) -> usize { self.cap }
}

impl<T> Drop for RawVec<T> {
    fn drop(&mut self) {
        if self.cap > 0 {
            let layout = Layout::array::<T>(self.cap).unwrap();
            // SAFETY: ptr was allocated with this layout; no elements are dropped here.
            unsafe { dealloc(self.ptr.as_ptr() as *mut u8, layout); }
        }
    }
}
```

## Pin and Self-Referential Structs

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

pub struct SelfReferential {
    data: i32,
    ptr: *const i32, // points to `self.data`
    _pin: PhantomPinned,
}

impl SelfReferential {
    pub fn new(data: i32) -> Pin<Box<Self>> {
        let mut boxed = Box::new(Self {
            data,
            ptr: std::ptr::null(),
            _pin: PhantomPinned,
        });
        // SAFETY: Box ensures the value won't move after pinning.
        let ptr = &boxed.data as *const i32;
        boxed.ptr = ptr;
        Pin::new(boxed)
    }

    pub fn get_data(self: Pin<&Self>) -> i32 {
        // SAFETY: ptr is known to be valid and initialized from new().
        unsafe { *self.ptr }
    }
}
```

## Transmute Alternatives

Try to avoid `transmute`. Prefer safer alternatives:

```rust
// Bad: opaque, no compile-time size check.
let bytes: [u8; 4] = std::mem::transmute(1234u32);

// Good: explicit with FromBytes / Pod traits (bytemuck).
let bytes: [u8; 4] = 1234u32.to_ne_bytes();

// Or for safe reinterpretation of slices:
let bytes: &[u8] = bytemuck::cast_slice(&[1.0f32, 2.0f32]);
```

## Union Access

```rust
#[repr(C)]
pub union IntOrFloat {
    pub int: i32,
    pub float: f32,
}

// SAFETY: reading the union requires knowing which field was last written.
// Store a discriminant alongside the union to track this.
pub struct TypedValue {
    value: IntOrFloat,
    is_int: bool,
}

impl TypedValue {
    pub fn new_int(val: i32) -> Self {
        Self { value: IntOrFloat { int: val }, is_int: true }
    }

    pub fn as_int(&self) -> Option<i32> {
        if self.is_int {
            // SAFETY: discriminant guarantees this field was last written.
            Some(unsafe { self.value.int })
        } else {
            None
        }
    }
}
```

## Automated Soundness Checks: Miri

Always validate unsafe code using Miri, Rust's Undefined Behavior interpreter.

```bash
# Run tests inside Miri to catch invalid pointer accesses, alignments, and resource leaks
cargo miri test

# Test with specific flags
cargo miri test -- test_unsafe
```

```rust
#[cfg(miri)]
#[test]
fn test_ring_buffer_miri() {
    let mut buf = RingBuffer::with_capacity(4);
    buf.push(1).unwrap();
    buf.push(2).unwrap();
    assert_eq!(buf.pop(), Some(1));
    assert_eq!(buf.pop(), Some(2));
    assert_eq!(buf.pop(), None);
}
```

## Unsafe Review Workflow

1. Identify the safe abstraction boundary.
2. List every invariant unsafe code relies on.
3. Ensure safe callers cannot violate those invariants.
4. Add `// SAFETY:` comments at each unsafe block.
5. Run tests under Miri when feasible.
6. Minimize unsafe surface area.
7. Document what must hold for each unsafe operation.

## Pointer Rules

Raw pointers can be null, dangling, unaligned, or aliased. Convert to references only after proving validity.

```rust
pub fn get(slice: &[u8], index: usize) -> Option<u8> {
    if index < slice.len() {
        // SAFETY: index is bounds-checked and pointer comes from a valid slice.
        Some(unsafe { *slice.as_ptr().add(index) })
    } else {
        None
    }
}
```

## FFI Boundaries

- Use `#[repr(C)]` for C-facing structs.
- Never let Rust panics cross `extern "C"` boundaries.
- Document ownership transfer for every pointer.
- Accept null only when the API explicitly supports it.
- Provide matching allocation/free functions when Rust owns memory.
- Use `extern "C"` for callbacks passed to C.
- Use `catch_unwind` to prevent panics from crossing FFI boundaries.

```rust
use std::panic::catch_unwind;

/// # Safety
/// `data` must be a valid, non-null pointer to a C string.
pub extern "C" fn callback(data: *const libc::c_char) -> i32 {
    let result = catch_unwind(|| {
        let c_str = unsafe { std::ffi::CStr::from_ptr(data) };
        let s = c_str.to_str().unwrap_or("");
        process(s)
    });

    match result {
        Ok(val) => val,
        Err(_) => -1, // don't let the panic cross FFI
    }
}
```

## Unsafe Traits

Unsafe traits mean implementors must uphold invariants the compiler cannot verify.

```rust
/// # Safety
/// Implementors must guarantee the returned pointer is valid for `len` bytes.
pub unsafe trait ByteSource {
    fn ptr(&self) -> *const u8;
    fn len(&self) -> usize;
}
```

## Concurrency Hazards

Manually implementing `Send` or `Sync` is a major soundness claim. Check interior mutability, aliasing, thread-affinity, and foreign handles.

```rust
struct MyRc<T> {
    inner: *mut Inner<T>,
}

// SAFETY: MyRc has exclusive access semantics for mutation.
unsafe impl<T: Send> Send for MyRc<T> {}
unsafe impl<T: Sync> Sync for MyRc<T> {}
```

## Anti-Patterns

```rust
// Bad: transmute when safe alternatives exist.
let val: u64 = transmute(float_val);

// Good: use to_bits / from_bits for primitive transmutes.
let val: u64 = float_val.to_bits();
```

```rust
// Bad: creating &mut T from raw pointers while aliases exist.
let ptr = &mut val as *mut i32;
let r1 = unsafe { &mut *ptr };
let r2 = unsafe { &mut *ptr }; // UB! Two mutable references to the same value
```

```rust
// Bad: safe functions with unchecked pointer offsets.
pub fn read_at(slice: &[u8], offset: usize) -> u8 {
    unsafe { *slice.as_ptr().add(offset) } // no bounds check
}

// Good: validate before unsafe.
pub fn read_at(slice: &[u8], offset: usize) -> Option<u8> {
    if offset < slice.len() {
        Some(unsafe { *slice.as_ptr().add(offset) })
    } else {
        None
    }
}
```

- Missing safety docs on `unsafe fn`.
- Treating Miri success as a proof of correctness (Miri doesn't catch logic bugs).
- `mem::zeroed()` where `MaybeUninit` should be used.
- Forgetting to drop elements when implementing a custom collection.

## Review Prompt

For unsafe Rust, require invariants, safety comments, safe-boundary proof, Miri coverage, FFI ownership docs, and justification for each unsafe operation.

## Production Readiness Checklist

- Inputs are validated at boundaries.
- Errors preserve enough context for debugging.
- Expensive work is measured or bounded.
- Examples avoid hidden global state.
- Security and privacy assumptions are stated.
- Tests cover edge cases and failure paths.
- All unsafe blocks have `// SAFETY:` comments.
- Safe public API cannot trigger UB for any input.
- Miri tests cover the unsafe code paths.
- FFI boundary has panic guards and ownership documentation.
- Custom collections implement Drop correctly for all elements.

## References

- [Rustonomicon](https://doc.rust-lang.org/nomicon/)
- [Miri](https://github.com/rust-lang/miri)
- [Rust Reference: behavior considered undefined](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)
- [Unsafe Code Guidelines](https://rust-lang.github.io/unsafe-code-guidelines/)
- [Pin and self-referential structs](https://doc.rust-lang.org/std/pin/index.html)
