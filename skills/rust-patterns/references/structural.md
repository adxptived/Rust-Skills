# Structural Patterns in Rust

Patterns for composing types and managing relationships.

## Newtype Pattern

Wrap a type to create distinct type with same representation:

```rust
// Prevent mixing IDs
pub struct UserId(pub u64);
pub struct OrderId(pub u64);

// Can't accidentally use OrderId where UserId expected
fn get_user(id: UserId) -> User { ... }
fn get_order(id: OrderId) -> Order { ... }

// With validation
pub struct Email(String);

impl Email {
    pub fn new(s: impl AsRef<str>) -> Result<Self, EmailError> {
        let s = s.as_ref();
        if !s.contains('@') || !s.contains('.') {
            return Err(EmailError::Invalid);
        }
        Ok(Self(s.to_string()))
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
    
    pub fn into_inner(self) -> String {
        self.0
    }
}

// Implement Deref for transparent access
use std::ops::Deref;

impl Deref for Email {
    type Target = str;
    
    fn deref(&self) -> &str {
        &self.0
    }
}

// Now email.len(), email.contains("@") work
let email = Email::new("user@example.com")?;
println!("Length: {}", email.len());
```

### Newtype for Units

```rust
pub struct Meters(f64);
pub struct Feet(f64);
pub struct Seconds(f64);

impl Meters {
    pub fn to_feet(self) -> Feet {
        Feet(self.0 * 3.28084)
    }
}

impl std::ops::Add for Meters {
    type Output = Self;
    
    fn add(self, other: Self) -> Self {
        Meters(self.0 + other.0)
    }
}

// Can't accidentally add Meters + Feet
let distance = Meters(100.0) + Meters(50.0);  // OK
// let wrong = Meters(100.0) + Feet(50.0);    // Compile error!
```

## Wrapper / Decorator Pattern

Add functionality to existing types:

```rust
pub struct LoggingDb<D: Database> {
    inner: D,
    logger: Logger,
}

impl<D: Database> LoggingDb<D> {
    pub fn new(db: D, logger: Logger) -> Self {
        Self { inner: db, logger }
    }
}

impl<D: Database> Database for LoggingDb<D> {
    fn query(&self, sql: &str) -> Result<Rows, DbError> {
        self.logger.debug(&format!("Query: {}", sql));
        let start = Instant::now();
        let result = self.inner.query(sql);
        self.logger.debug(&format!("Query took {:?}", start.elapsed()));
        result
    }
    
    fn execute(&self, sql: &str) -> Result<usize, DbError> {
        self.logger.debug(&format!("Execute: {}", sql));
        self.inner.execute(sql)
    }
}

// Stack decorators
let db = PostgresDb::connect(url)?;
let db = LoggingDb::new(db, logger);
let db = CachingDb::new(db, cache);
let db = RetryingDb::new(db, 3);
```

## Adapter Pattern

Convert one interface to another:

```rust
// External library's interface
pub struct LegacyPrinter {
    pub fn print_document(&self, doc: LegacyDocument);
}

// Our interface
pub trait Printer {
    fn print(&self, content: &str);
}

// Adapter
pub struct LegacyPrinterAdapter {
    legacy: LegacyPrinter,
}

impl LegacyPrinterAdapter {
    pub fn new(legacy: LegacyPrinter) -> Self {
        Self { legacy }
    }
}

impl Printer for LegacyPrinterAdapter {
    fn print(&self, content: &str) {
        let doc = LegacyDocument::from_string(content);
        self.legacy.print_document(doc);
    }
}

// Now works with our Printer trait
fn print_report(printer: &impl Printer, report: &Report) {
    printer.print(&report.to_string());
}
```

## Facade Pattern

Simplify complex subsystems:

```rust
pub struct MediaConverter {
    audio_encoder: AudioEncoder,
    video_encoder: VideoEncoder,
    muxer: Muxer,
    metadata: MetadataWriter,
}

impl MediaConverter {
    pub fn new() -> Self {
        Self {
            audio_encoder: AudioEncoder::new(Codec::AAC),
            video_encoder: VideoEncoder::new(Codec::H264),
            muxer: Muxer::new(Container::MP4),
            metadata: MetadataWriter::new(),
        }
    }
    
    // Simple interface hides complexity
    pub fn convert(&self, input: &Path, output: &Path) -> Result<(), ConvertError> {
        let video = self.video_encoder.encode(input)?;
        let audio = self.audio_encoder.encode(input)?;
        let muxed = self.muxer.mux(video, audio)?;
        self.metadata.write(output, &muxed)?;
        Ok(())
    }
    
    pub fn convert_with_options(
        &self,
        input: &Path,
        output: &Path,
        options: ConvertOptions,
    ) -> Result<(), ConvertError> {
        // More control when needed
        let video = self.video_encoder
            .with_bitrate(options.video_bitrate)
            .with_resolution(options.resolution)
            .encode(input)?;
        // ...
    }
}
```

## Composite Pattern

Tree structures with uniform interface:

```rust
pub trait Component {
    fn render(&self) -> String;
    fn add(&mut self, _: Box<dyn Component>) {}
    fn remove(&mut self, _: usize) {}
}

pub struct Leaf {
    content: String,
}

impl Component for Leaf {
    fn render(&self) -> String {
        self.content.clone()
    }
}

pub struct Container {
    children: Vec<Box<dyn Component>>,
}

impl Component for Container {
    fn render(&self) -> String {
        self.children
            .iter()
            .map(|c| c.render())
            .collect::<Vec<_>>()
            .join("\n")
    }
    
    fn add(&mut self, component: Box<dyn Component>) {
        self.children.push(component);
    }
    
    fn remove(&mut self, index: usize) {
        self.children.remove(index);
    }
}

// Usage - same interface for leaf and container
let mut root = Container { children: vec![] };
root.add(Box::new(Leaf { content: "Header".into() }));
root.add(Box::new(Container {
    children: vec![
        Box::new(Leaf { content: "Item 1".into() }),
        Box::new(Leaf { content: "Item 2".into() }),
    ],
}));
println!("{}", root.render());
```

### Enum-Based Composite

More idiomatic Rust:

```rust
pub enum Element {
    Text(String),
    Container(Vec<Element>),
    Link { text: String, url: String },
}

impl Element {
    pub fn render(&self) -> String {
        match self {
            Element::Text(s) => s.clone(),
            Element::Container(children) => {
                children.iter()
                    .map(|c| c.render())
                    .collect::<Vec<_>>()
                    .join("\n")
            }
            Element::Link { text, url } => {
                format!("[{}]({})", text, url)
            }
        }
    }
}
```

## Extension Trait Pattern

Add methods to external types:

```rust
pub trait StringExt {
    fn is_blank(&self) -> bool;
    fn truncate_ellipsis(&self, max_len: usize) -> Cow<'_, str>;
    fn to_title_case(&self) -> String;
}

impl StringExt for str {
    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
    
    fn truncate_ellipsis(&self, max_len: usize) -> Cow<'_, str> {
        if self.len() <= max_len {
            Cow::Borrowed(self)
        } else {
            Cow::Owned(format!("{}...", &self[..max_len.saturating_sub(3)]))
        }
    }
    
    fn to_title_case(&self) -> String {
        self.split_whitespace()
            .map(|word| {
                let mut chars = word.chars();
                match chars.next() {
                    None => String::new(),
                    Some(c) => c.to_uppercase().collect::<String>() + chars.as_str(),
                }
            })
            .collect::<Vec<_>>()
            .join(" ")
    }
}

// Usage
"  ".is_blank();                           // true
"hello world".truncate_ellipsis(8);        // "hello..."
"hello world".to_title_case();             // "Hello World"
```

### Extension Traits for Results

```rust
pub trait ResultExt<T, E> {
    fn log_error(self, msg: &str) -> Self;
    fn ignore_error(self) -> Option<T>;
}

impl<T, E: std::fmt::Debug> ResultExt<T, E> for Result<T, E> {
    fn log_error(self, msg: &str) -> Self {
        if let Err(ref e) = self {
            eprintln!("{}: {:?}", msg, e);
        }
        self
    }
    
    fn ignore_error(self) -> Option<T> {
        self.ok()
    }
}

// Usage
let config = load_config()
    .log_error("Failed to load config")
    .unwrap_or_default();
```

## Flyweight Pattern

Share common data between objects:

```rust
use std::sync::Arc;

pub struct Character {
    // Shared (flyweight)
    font: Arc<Font>,
    // Unique per instance
    position: (i32, i32),
    char: char,
}

pub struct FontCache {
    fonts: HashMap<String, Arc<Font>>,
}

impl FontCache {
    pub fn get(&mut self, name: &str) -> Arc<Font> {
        self.fonts
            .entry(name.to_string())
            .or_insert_with(|| Arc::new(Font::load(name)))
            .clone()
    }
}

// Many characters share same Font
let mut cache = FontCache::new();
let font = cache.get("Arial");
let chars: Vec<Character> = text.chars()
    .enumerate()
    .map(|(i, c)| Character {
        font: font.clone(),  // Cheap Arc clone
        position: (i as i32 * 10, 0),
        char: c,
    })
    .collect();
```

## Bridge Pattern

Separate abstraction from implementation:

```rust
// Implementation hierarchy
pub trait Renderer {
    fn render_circle(&self, x: f32, y: f32, radius: f32);
    fn render_rect(&self, x: f32, y: f32, w: f32, h: f32);
}

pub struct OpenGLRenderer;
pub struct VulkanRenderer;

impl Renderer for OpenGLRenderer {
    fn render_circle(&self, x: f32, y: f32, radius: f32) {
        println!("OpenGL circle at ({}, {}) r={}", x, y, radius);
    }
    fn render_rect(&self, x: f32, y: f32, w: f32, h: f32) {
        println!("OpenGL rect at ({}, {}) {}x{}", x, y, w, h);
    }
}

// Abstraction hierarchy
pub trait Shape {
    fn draw(&self, renderer: &dyn Renderer);
}

pub struct Circle {
    x: f32,
    y: f32,
    radius: f32,
}

impl Shape for Circle {
    fn draw(&self, renderer: &dyn Renderer) {
        renderer.render_circle(self.x, self.y, self.radius);
    }
}

// Any Shape can use any Renderer
let shapes: Vec<Box<dyn Shape>> = vec![...];
let renderer = OpenGLRenderer;
for shape in &shapes {
    shape.draw(&renderer);
}
```
