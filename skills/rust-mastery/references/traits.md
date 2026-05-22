# Traits and Generics

Complete guide to Rust's trait system, generics, and polymorphism.

## Table of Contents
1. [Trait Basics](#trait-basics)
2. [Standard Traits](#standard-traits)
3. [Generics](#generics)
4. [Trait Objects](#trait-objects)
5. [Advanced Patterns](#advanced-patterns)

## Trait Basics

Traits define shared behavior. Similar to interfaces in other languages.

```rust
trait Summary {
    // Required method
    fn summarize(&self) -> String;
    
    // Default implementation
    fn preview(&self) -> String {
        format!("Read more: {}", self.summarize())
    }
}

struct Article {
    title: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}", self.title, &self.content[..50])
    }
    // preview() uses default implementation
}
```

### Associated Types

For traits with a type that implementors choose:

```rust
trait Iterator {
    type Item; // Associated type
    
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32; // Counter yields u32
    
    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}
```

### Associated Constants

```rust
trait Float {
    const ZERO: Self;
    const ONE: Self;
}

impl Float for f32 {
    const ZERO: f32 = 0.0;
    const ONE: f32 = 1.0;
}
```

## Standard Traits

### Derivable Traits

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]
struct Point {
    x: i32,
    y: i32,
}
```

| Trait | Derive? | Purpose |
|-------|---------|---------|
| `Debug` | Yes | `{:?}` formatting |
| `Clone` | Yes | `.clone()` method |
| `Copy` | Yes | Implicit copy (bitwise) |
| `Default` | Yes | `Default::default()` |
| `PartialEq` | Yes | `==` and `!=` |
| `Eq` | Yes | Reflexive equality marker |
| `PartialOrd` | Yes | `<`, `>`, `<=`, `>=` |
| `Ord` | Yes | Total ordering |
| `Hash` | Yes | Hashing for collections |

### Display vs Debug

```rust
use std::fmt;

struct Point { x: i32, y: i32 }

// Debug: developer-facing, derivable
impl fmt::Debug for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}

// Display: user-facing, must implement manually
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

let p = Point { x: 1, y: 2 };
println!("{:?}", p); // Point { x: 1, y: 2 }
println!("{}", p);   // (1, 2)
```

### From and Into

```rust
struct Wrapper(i32);

impl From<i32> for Wrapper {
    fn from(value: i32) -> Self {
        Wrapper(value)
    }
}

// Into is automatically implemented when From is
let w: Wrapper = 42.into();
let w: Wrapper = Wrapper::from(42);

// Useful for flexible APIs
fn process(value: impl Into<Wrapper>) {
    let w = value.into();
    // ...
}

process(42);           // Converted via Into
process(Wrapper(42));  // Already correct type
```

### AsRef and AsMut

Cheap reference conversions:

```rust
// Accept anything that can be viewed as &str
fn print_message(msg: impl AsRef<str>) {
    println!("{}", msg.as_ref());
}

print_message("hello");              // &str
print_message(String::from("world")); // String
print_message(&String::from("!")); // &String

// For paths
fn read_file(path: impl AsRef<Path>) -> io::Result<String> {
    std::fs::read_to_string(path.as_ref())
}

read_file("config.txt");
read_file(PathBuf::from("config.txt"));
```

### Deref and DerefMut

Smart pointer behavior:

```rust
use std::ops::{Deref, DerefMut};

struct MyBox<T>(T);

impl<T> Deref for MyBox<T> {
    type Target = T;
    
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

// Deref coercion
let b = MyBox(String::from("hello"));
let s: &str = &b; // MyBox<String> -> &String -> &str
```

### Drop

Cleanup when value goes out of scope:

```rust
struct TempFile(PathBuf);

impl Drop for TempFile {
    fn drop(&mut self) {
        println!("Cleaning up {:?}", self.0);
        let _ = std::fs::remove_file(&self.0);
    }
}

{
    let _temp = TempFile(PathBuf::from("/tmp/data"));
    // Use temp file...
} // TempFile::drop called automatically
```

### Iterator

```rust
struct Range {
    start: i32,
    end: i32,
}

impl Iterator for Range {
    type Item = i32;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.start < self.end {
            let current = self.start;
            self.start += 1;
            Some(current)
        } else {
            None
        }
    }
}

// Many methods come free from Iterator trait
let range = Range { start: 0, end: 5 };
let sum: i32 = range.sum();
```

## Generics

### Generic Functions

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

### Generic Structs

```rust
struct Point<T> {
    x: T,
    y: T,
}

// Different types for x and y
struct Point2<T, U> {
    x: T,
    y: U,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

// Implement only for specific types
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

### Trait Bounds

```rust
// Single bound
fn print_debug<T: Debug>(value: T) {
    println!("{:?}", value);
}

// Multiple bounds
fn compare_and_display<T: PartialOrd + Display>(a: T, b: T) {
    if a > b {
        println!("{} > {}", a, b);
    }
}

// Where clause (cleaner for complex bounds)
fn complex<T, U>(t: T, u: U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // ...
}

// impl Trait syntax
fn make_iterator() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
}
```

### Conditional Implementation

```rust
struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

// Only implement for comparable types
impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("Largest: {}", self.x);
        } else {
            println!("Largest: {}", self.y);
        }
    }
}

// Blanket implementation
impl<T: Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{}", self)
    }
}
```

## Trait Objects

Dynamic dispatch via `dyn Trait`:

```rust
trait Draw {
    fn draw(&self);
}

struct Button { label: String }
struct TextField { text: String }

impl Draw for Button {
    fn draw(&self) { println!("Button: {}", self.label); }
}

impl Draw for TextField {
    fn draw(&self) { println!("TextField: {}", self.text); }
}

// Heterogeneous collection
let components: Vec<Box<dyn Draw>> = vec![
    Box::new(Button { label: "OK".to_string() }),
    Box::new(TextField { text: "Enter name".to_string() }),
];

for component in &components {
    component.draw(); // Dynamic dispatch
}
```

### Static vs Dynamic Dispatch

| Static (Generics) | Dynamic (Trait Objects) |
|-------------------|------------------------|
| `impl Trait` / `<T: Trait>` | `dyn Trait` |
| Monomorphization at compile time | Vtable lookup at runtime |
| No runtime cost | Small overhead |
| Larger binary (code duplication) | Smaller binary |
| Must know type at compile time | Heterogeneous collections |

### Object Safety

A trait is object-safe if:
- All methods have `Self: Sized` bound, OR
- Methods satisfy:
  - Don't return `Self`
  - Don't have generic type parameters
  - No `Self` in argument position (except receiver)

```rust
// NOT object-safe (returns Self)
trait Clone {
    fn clone(&self) -> Self;
}

// Object-safe
trait Draw {
    fn draw(&self);
}

// Making non-object-safe traits work
trait MyClone {
    fn clone_box(&self) -> Box<dyn MyClone>;
}
```

## Advanced Patterns

### Supertraits

```rust
trait Animal {
    fn name(&self) -> &str;
}

// Dog requires Animal
trait Dog: Animal {
    fn bark(&self);
}

struct Labrador { name: String }

impl Animal for Labrador {
    fn name(&self) -> &str { &self.name }
}

impl Dog for Labrador {
    fn bark(&self) { println!("Woof!"); }
}
```

### Extension Traits

Add methods to existing types:

```rust
trait StringExt {
    fn is_blank(&self) -> bool;
}

impl StringExt for str {
    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
}

// Now all &str have .is_blank()
"   ".is_blank() // true
```

### Marker Traits

```rust
// Empty trait used to mark types
trait Marker {}

struct Safe;
struct Unsafe;

impl Marker for Safe {}

fn process<T: Marker>(item: T) {
    // Only accepts types marked with Marker
}

process(Safe);   // OK
// process(Unsafe); // Error: Unsafe doesn't implement Marker
```

### Sealed Traits

Prevent external implementation:

```rust
mod private {
    pub trait Sealed {}
}

pub trait MyTrait: private::Sealed {
    fn method(&self);
}

// Only types you impl Sealed for can impl MyTrait
impl private::Sealed for MyType {}
impl MyTrait for MyType {
    fn method(&self) { ... }
}
```

### Fully Qualified Syntax

When trait methods conflict:

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) { println!("Captain speaking"); }
}

impl Wizard for Human {
    fn fly(&self) { println!("Wingardium Leviosa!"); }
}

impl Human {
    fn fly(&self) { println!("Waving arms"); }
}

let person = Human;
person.fly();           // "Waving arms" (inherent method)
Pilot::fly(&person);    // "Captain speaking"
Wizard::fly(&person);   // "Wingardium Leviosa!"

// For associated functions (no self)
<Human as Pilot>::fly(&person);
```
