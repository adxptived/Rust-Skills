# PhantomData and Marker Types

`PhantomData` tells the compiler about type or lifetime relationships not stored as fields.

## Marker States

```rust
pub struct Draft;
pub struct Published;

pub struct Document<State> {
    title: String,
    _state: std::marker::PhantomData<State>,
}
```

The marker state exists at compile time and has no runtime size.

## Lifetime Markers

Use lifetime markers when a type logically borrows data through raw pointers or handles.

```rust
pub struct SliceView<'a, T> {
    ptr: *const T,
    len: usize,
    _marker: std::marker::PhantomData<&'a T>,
}
```

This prevents the view from outliving the source allocation.

## Ownership Markers

`PhantomData<T>` means the type acts as if it owns `T`. This affects drop check, variance, and auto traits.

```rust
pub struct OwningHandle<T> {
    raw: *mut T,
    _owns: std::marker::PhantomData<T>,
}
```

## Variance

Marker choice matters:

- `PhantomData<T>`: owns `T`.
- `PhantomData<&'a T>`: shared borrow.
- `PhantomData<&'a mut T>`: exclusive borrow.
- `PhantomData<fn(T)>`: contravariant marker.

## Rule

Use `PhantomData` only when the compiler cannot see the relationship from real fields.
