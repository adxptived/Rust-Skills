# Behavioral Patterns in Rust

Patterns for managing algorithms, state, and object interactions.

## State Machine Pattern

### Enum-Based State Machine

Most common approach in Rust:

```rust
pub enum OrderState {
    Pending,
    Confirmed { confirmed_at: DateTime<Utc> },
    Shipped { tracking: String, shipped_at: DateTime<Utc> },
    Delivered { delivered_at: DateTime<Utc> },
    Cancelled { reason: String },
}

pub struct Order {
    id: OrderId,
    items: Vec<Item>,
    state: OrderState,
}

impl Order {
    pub fn confirm(&mut self) -> Result<(), OrderError> {
        match &self.state {
            OrderState::Pending => {
                self.state = OrderState::Confirmed {
                    confirmed_at: Utc::now(),
                };
                Ok(())
            }
            _ => Err(OrderError::InvalidTransition),
        }
    }
    
    pub fn ship(&mut self, tracking: String) -> Result<(), OrderError> {
        match &self.state {
            OrderState::Confirmed { .. } => {
                self.state = OrderState::Shipped {
                    tracking,
                    shipped_at: Utc::now(),
                };
                Ok(())
            }
            _ => Err(OrderError::InvalidTransition),
        }
    }
    
    pub fn deliver(&mut self) -> Result<(), OrderError> {
        match &self.state {
            OrderState::Shipped { .. } => {
                self.state = OrderState::Delivered {
                    delivered_at: Utc::now(),
                };
                Ok(())
            }
            _ => Err(OrderError::InvalidTransition),
        }
    }
    
    pub fn cancel(&mut self, reason: String) -> Result<(), OrderError> {
        match &self.state {
            OrderState::Pending | OrderState::Confirmed { .. } => {
                self.state = OrderState::Cancelled { reason };
                Ok(())
            }
            _ => Err(OrderError::InvalidTransition),
        }
    }
}
```

### Typestate Pattern

Compile-time state enforcement:

```rust
use std::marker::PhantomData;

// State markers
pub struct Draft;
pub struct InReview;
pub struct Published;
pub struct Archived;

pub struct Article<State> {
    title: String,
    content: String,
    _state: PhantomData<State>,
}

impl Article<Draft> {
    pub fn new(title: String) -> Self {
        Self {
            title,
            content: String::new(),
            _state: PhantomData,
        }
    }
    
    pub fn edit(&mut self, content: String) {
        self.content = content;
    }
    
    pub fn submit_for_review(self) -> Article<InReview> {
        Article {
            title: self.title,
            content: self.content,
            _state: PhantomData,
        }
    }
}

impl Article<InReview> {
    pub fn approve(self) -> Article<Published> {
        Article {
            title: self.title,
            content: self.content,
            _state: PhantomData,
        }
    }
    
    pub fn reject(self, _feedback: &str) -> Article<Draft> {
        Article {
            title: self.title,
            content: self.content,
            _state: PhantomData,
        }
    }
}

impl Article<Published> {
    pub fn archive(self) -> Article<Archived> {
        Article {
            title: self.title,
            content: self.content,
            _state: PhantomData,
        }
    }
    
    // Can't edit published articles!
}

// Usage
let article = Article::new("My Post".into());
article.edit("Content...".into());  // OK in Draft
let article = article.submit_for_review();
// article.edit(...);  // Compile error! Not in Draft state
let article = article.approve();
// article.reject(...);  // Compile error! Not in InReview state
```

## Strategy Pattern

Swappable algorithms:

```rust
pub trait SortStrategy {
    fn sort<T: Ord>(&self, data: &mut [T]);
}

pub struct QuickSort;
pub struct MergeSort;
pub struct InsertionSort;

impl SortStrategy for QuickSort {
    fn sort<T: Ord>(&self, data: &mut [T]) {
        data.sort_unstable();
    }
}

impl SortStrategy for MergeSort {
    fn sort<T: Ord>(&self, data: &mut [T]) {
        data.sort();
    }
}

impl SortStrategy for InsertionSort {
    fn sort<T: Ord>(&self, data: &mut [T]) {
        for i in 1..data.len() {
            let mut j = i;
            while j > 0 && data[j - 1] > data[j] {
                data.swap(j - 1, j);
                j -= 1;
            }
        }
    }
}

pub struct Sorter<S: SortStrategy> {
    strategy: S,
}

impl<S: SortStrategy> Sorter<S> {
    pub fn new(strategy: S) -> Self {
        Self { strategy }
    }
    
    pub fn sort<T: Ord>(&self, data: &mut [T]) {
        self.strategy.sort(data);
    }
}

// Choose strategy at compile time
let sorter = Sorter::new(QuickSort);
sorter.sort(&mut data);
```

### Closure-Based Strategy

```rust
pub struct Processor<F>
where
    F: Fn(&str) -> String,
{
    transform: F,
}

impl<F: Fn(&str) -> String> Processor<F> {
    pub fn new(transform: F) -> Self {
        Self { transform }
    }
    
    pub fn process(&self, input: &str) -> String {
        (self.transform)(input)
    }
}

// Flexible strategies via closures
let uppercase = Processor::new(|s| s.to_uppercase());
let reverse = Processor::new(|s| s.chars().rev().collect());
```

## Observer Pattern

### Channel-Based

```rust
use tokio::sync::broadcast;

pub struct EventBus {
    sender: broadcast::Sender<Event>,
}

impl EventBus {
    pub fn new() -> Self {
        let (sender, _) = broadcast::channel(100);
        Self { sender }
    }
    
    pub fn subscribe(&self) -> broadcast::Receiver<Event> {
        self.sender.subscribe()
    }
    
    pub fn publish(&self, event: Event) {
        let _ = self.sender.send(event);
    }
}

// Subscriber
async fn handle_events(mut rx: broadcast::Receiver<Event>) {
    while let Ok(event) = rx.recv().await {
        match event {
            Event::UserCreated(user) => println!("New user: {}", user.name),
            Event::OrderPlaced(order) => println!("New order: {}", order.id),
        }
    }
}
```

### Callback-Based

```rust
pub struct Observable<T> {
    value: T,
    listeners: Vec<Box<dyn Fn(&T)>>,
}

impl<T> Observable<T> {
    pub fn new(value: T) -> Self {
        Self {
            value,
            listeners: Vec::new(),
        }
    }
    
    pub fn subscribe(&mut self, callback: impl Fn(&T) + 'static) {
        self.listeners.push(Box::new(callback));
    }
    
    pub fn set(&mut self, value: T) {
        self.value = value;
        self.notify();
    }
    
    fn notify(&self) {
        for listener in &self.listeners {
            listener(&self.value);
        }
    }
}
```

## Command Pattern

Encapsulate operations as objects:

```rust
pub trait Command {
    fn execute(&mut self);
    fn undo(&mut self);
}

pub struct InsertText {
    document: Rc<RefCell<Document>>,
    position: usize,
    text: String,
}

impl Command for InsertText {
    fn execute(&mut self) {
        self.document.borrow_mut().insert(self.position, &self.text);
    }
    
    fn undo(&mut self) {
        self.document.borrow_mut().delete(self.position, self.text.len());
    }
}

pub struct DeleteText {
    document: Rc<RefCell<Document>>,
    position: usize,
    deleted: String,
}

impl Command for DeleteText {
    fn execute(&mut self) {
        let doc = self.document.borrow();
        self.deleted = doc.get_range(self.position, self.deleted.len());
        drop(doc);
        self.document.borrow_mut().delete(self.position, self.deleted.len());
    }
    
    fn undo(&mut self) {
        self.document.borrow_mut().insert(self.position, &self.deleted);
    }
}

pub struct CommandHistory {
    history: Vec<Box<dyn Command>>,
    position: usize,
}

impl CommandHistory {
    pub fn execute(&mut self, mut cmd: Box<dyn Command>) {
        cmd.execute();
        self.history.truncate(self.position);
        self.history.push(cmd);
        self.position += 1;
    }
    
    pub fn undo(&mut self) {
        if self.position > 0 {
            self.position -= 1;
            self.history[self.position].undo();
        }
    }
    
    pub fn redo(&mut self) {
        if self.position < self.history.len() {
            self.history[self.position].execute();
            self.position += 1;
        }
    }
}
```

## Visitor Pattern

### Enum Dispatch (Preferred)

```rust
pub enum Expr {
    Literal(i64),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
    Neg(Box<Expr>),
}

impl Expr {
    pub fn eval(&self) -> i64 {
        match self {
            Expr::Literal(n) => *n,
            Expr::Add(a, b) => a.eval() + b.eval(),
            Expr::Mul(a, b) => a.eval() * b.eval(),
            Expr::Neg(e) => -e.eval(),
        }
    }
    
    pub fn to_string(&self) -> String {
        match self {
            Expr::Literal(n) => n.to_string(),
            Expr::Add(a, b) => format!("({} + {})", a.to_string(), b.to_string()),
            Expr::Mul(a, b) => format!("({} * {})", a.to_string(), b.to_string()),
            Expr::Neg(e) => format!("(-{})", e.to_string()),
        }
    }
}
```

### Traditional Visitor

```rust
pub trait Visitor {
    fn visit_literal(&mut self, value: i64);
    fn visit_add(&mut self, left: &Expr, right: &Expr);
    fn visit_mul(&mut self, left: &Expr, right: &Expr);
}

impl Expr {
    pub fn accept(&self, visitor: &mut dyn Visitor) {
        match self {
            Expr::Literal(n) => visitor.visit_literal(*n),
            Expr::Add(a, b) => visitor.visit_add(a, b),
            Expr::Mul(a, b) => visitor.visit_mul(a, b),
        }
    }
}

pub struct Evaluator {
    result: i64,
}

impl Visitor for Evaluator {
    fn visit_literal(&mut self, value: i64) {
        self.result = value;
    }
    
    fn visit_add(&mut self, left: &Expr, right: &Expr) {
        left.accept(self);
        let l = self.result;
        right.accept(self);
        self.result = l + self.result;
    }
    
    fn visit_mul(&mut self, left: &Expr, right: &Expr) {
        left.accept(self);
        let l = self.result;
        right.accept(self);
        self.result = l * self.result;
    }
}
```

## Iterator Pattern

```rust
pub struct Counter {
    current: u64,
    max: u64,
}

impl Counter {
    pub fn new(max: u64) -> Self {
        Self { current: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u64;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.current < self.max {
            let result = self.current;
            self.current += 1;
            Some(result)
        } else {
            None
        }
    }
}

// Usage
for n in Counter::new(10) {
    println!("{}", n);
}

let sum: u64 = Counter::new(100).sum();
```

### IntoIterator

```rust
pub struct Grid {
    cells: Vec<Vec<Cell>>,
}

impl<'a> IntoIterator for &'a Grid {
    type Item = &'a Cell;
    type IntoIter = GridIter<'a>;
    
    fn into_iter(self) -> Self::IntoIter {
        GridIter {
            grid: self,
            row: 0,
            col: 0,
        }
    }
}

pub struct GridIter<'a> {
    grid: &'a Grid,
    row: usize,
    col: usize,
}

impl<'a> Iterator for GridIter<'a> {
    type Item = &'a Cell;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.row >= self.grid.cells.len() {
            return None;
        }
        
        let cell = &self.grid.cells[self.row][self.col];
        
        self.col += 1;
        if self.col >= self.grid.cells[self.row].len() {
            self.col = 0;
            self.row += 1;
        }
        
        Some(cell)
    }
}

// Usage
for cell in &grid {
    process(cell);
}
```

## Chain of Responsibility

```rust
pub trait Handler {
    fn handle(&self, request: &Request) -> Option<Response>;
    fn set_next(&mut self, next: Box<dyn Handler>);
}

pub struct AuthHandler {
    next: Option<Box<dyn Handler>>,
}

impl Handler for AuthHandler {
    fn handle(&self, request: &Request) -> Option<Response> {
        if !request.is_authenticated() {
            return Some(Response::unauthorized());
        }
        self.next.as_ref()?.handle(request)
    }
    
    fn set_next(&mut self, next: Box<dyn Handler>) {
        self.next = Some(next);
    }
}

pub struct RateLimitHandler {
    next: Option<Box<dyn Handler>>,
    limiter: RateLimiter,
}

impl Handler for RateLimitHandler {
    fn handle(&self, request: &Request) -> Option<Response> {
        if !self.limiter.allow(request.client_id()) {
            return Some(Response::too_many_requests());
        }
        self.next.as_ref()?.handle(request)
    }
    
    fn set_next(&mut self, next: Box<dyn Handler>) {
        self.next = Some(next);
    }
}
```

### Middleware Chain

More common in web frameworks:

```rust
pub type Middleware = Box<dyn Fn(Request, Next) -> Response>;

pub struct Next<'a> {
    chain: &'a [Middleware],
}

impl<'a> Next<'a> {
    pub fn run(self, request: Request) -> Response {
        match self.chain.split_first() {
            Some((middleware, rest)) => {
                middleware(request, Next { chain: rest })
            }
            None => Response::not_found(),
        }
    }
}

// Usage
let middlewares: Vec<Middleware> = vec![
    Box::new(|req, next| {
        println!("Logging: {:?}", req);
        next.run(req)
    }),
    Box::new(|req, next| {
        if req.is_authenticated() {
            next.run(req)
        } else {
            Response::unauthorized()
        }
    }),
];
```
