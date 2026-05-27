# Resolving Borrow Checker Failures

Borrow checker errors are usually about ownership lifetime, aliasing, or mutation order.

## E0382: Use of Moved Value

A value was moved and then used again.

```rust
let data = vec![1, 2, 3];
consume(data);
// println!("{:?}", data); // moved
```

Fix by borrowing, cloning intentionally, or returning ownership.

```rust
fn inspect(data: &[i32]) { println!("{}", data.len()); }

let data = vec![1, 2, 3];
inspect(&data);
println!("{:?}", data);
```

## E0502: Mutable and Immutable Borrows Overlap

```rust
let first = values.first();
values.push(4); // cannot mutably borrow while `first` may be used
println!("{:?}", first);
```

Shorten the immutable borrow scope.

```rust
let first = values.first().copied();
values.push(4);
println!("{:?}", first);
```

## E0597: Borrowed Value Does Not Live Long Enough

Do not return references to local values.

```rust
fn bad<'a>() -> &'a str {
    let s = String::from("hello");
    // &s
    unimplemented_unreachable()
}
```

Return owned data or borrow from an input.

```rust
fn owned() -> String { String::from("hello") }
fn borrowed(input: &str) -> &str { input }
```

## Common Refactors

### Introduce Blocks

```rust
let len = {
    let name = user.name.as_str();
    name.len()
};
user.rename("new-name");
```

### Split Struct Borrows

Borrow separate fields instead of the whole struct.

```rust
let Config { host, port } = &config;
println!("{host}:{port}");
```

### Use `Option::take`

Move a field out safely while leaving `None` behind.

```rust
if let Some(job) = self.pending_job.take() {
    self.run(job);
}
```

### Clone at Boundaries

Clone small IDs, `Arc`, or config handles when it simplifies ownership. Do not clone large data in hot paths without measuring.

## Diagnostic Workflow

1. Identify who owns the value.
2. Identify where the borrow starts and ends.
3. Shorten borrows before adding clones.
4. Prefer borrowing inputs and owning outputs.
5. Use `Arc` only for shared ownership across tasks/threads.
