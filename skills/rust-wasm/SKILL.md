---
name: rust-wasm
description: |
  Rust WebAssembly development with wasm-bindgen, wasm-pack, web-sys, js-sys, and browser integration. Use when:
  - User compiles Rust to WebAssembly for web, workers, plugins, or edge runtimes
  - User uses wasm-bindgen, wasm-pack, web-sys, js-sys, gloo, trunk, or wasm-opt
  - User needs JS interop, typed bindings, memory transfer, or async browser APIs
  - User asks about WASM bundle size, panic hooks, console logging, or browser debugging
  - User integrates Rust WASM with React, Dioxus, Vite, npm packages, or web workers
  - User sees wasm32-unknown-unknown target errors or missing allocator/panic behavior
  - User needs shared memory, threads, atomics, or SIMD in WASM
  - User targets WASI or wasm32-wasip1 for server-side / edge runtimes
---

# Rust WebAssembly

Practical guide for shipping Rust code as WebAssembly: small boundaries, typed interop, predictable memory, and measurable bundle size.

## Quick Navigation

- **references/bindings.md** - wasm-bindgen, web-sys, JS types, async interop
- **references/bundling.md** - wasm-pack, Trunk, Vite, size optimization, deployment

## Golden Rules

1. Keep the Rust/JS boundary coarse-grained.
2. Pass bytes and IDs instead of many tiny objects.
3. Enable panic hooks in debug builds, not production hot paths.
4. Measure `.wasm` size after `wasm-opt`, not before.
5. Treat browser APIs as fallible external systems.
6. Use `cfg(target_family = "wasm")` for platform-specific code.
7. Avoid `wasm-bindgen` on hot loops — pass whole buffers.

## Project Setup

```bash
rustup target add wasm32-unknown-unknown
cargo install wasm-pack
cargo new image-core --lib
```

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
js-sys = "0.3"

[dependencies.web-sys]
version = "0.3"
features = ["console", "Window", "Document", "HtmlCanvasElement"]

[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
panic = "abort"
```

## Export a Small API

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Filter {
    strength: f32,
}

#[wasm_bindgen]
impl Filter {
    #[wasm_bindgen(constructor)]
    pub fn new(strength: f32) -> Self {
        Self { strength }
    }

    pub fn apply_rgba(&self, pixels: &mut [u8]) {
        for chunk in pixels.chunks_exact_mut(4) {
            let gray = (chunk[0] as f32 * 0.299
                + chunk[1] as f32 * 0.587
                + chunk[2] as f32 * 0.114) as u8;
            for channel in &mut chunk[..3] {
                *channel = ((*channel as f32 * (1.0 - self.strength))
                    + (gray as f32 * self.strength)) as u8;
            }
        }
    }
}
```

Use `&mut [u8]` for image/audio/data buffers. Avoid one exported call per pixel or row.

## Returning Complex Types

```rust
use serde::{Serialize, Deserialize};
use wasm_bindgen::prelude::*;

#[derive(Serialize, Deserialize)]
pub struct AnalysisResult {
    pub mean: f64,
    pub median: f64,
    pub std_dev: f64,
    pub count: usize,
}

#[wasm_bindgen]
pub fn analyze(data: &[f64]) -> JsValue {
    let mean = data.iter().sum::<f64>() / data.len() as f64;
    let result = AnalysisResult { mean, median: 0.0, std_dev: 0.0, count: data.len() };
    serde_wasm_bindgen::to_value(&result).unwrap()
}
```

Use `serde_wasm_bindgen` for complex return types, but avoid on hot paths where the serialization overhead matters.

## Accepting JS Callbacks

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn process_with_progress(
    data: &[u8],
    on_progress: &js_sys::Function,
) -> Result<(), JsValue> {
    let total = data.len();
    for (i, chunk) in data.chunks(1024).enumerate() {
        // process chunk
        let pct = (i as f64 / total as f64) * 100.0;
        let this = JsValue::NULL;
        let args = js_sys::Array::of1(&JsValue::from_f64(pct));
        on_progress.call(&this, &args)?;
    }
    Ok(())
}
```

For hot paths, prefer passing a channel or buffer over function callbacks to avoid per-call overhead.

## Wasm Threads (Shared Memory)

```toml
# Cargo.toml additional settings for threading
[package.metadata.wasm-pack.profile.release]
wasm-opt = false  # wasm-opt breaks shared memory
```

```rust
// Enable shared memory in the module
#[wasm_bindgen]
pub fn init_pool(num_workers: usize) -> Result<(), JsValue> {
    console_error_panic_hook::set_once();
    // Use wasm-bindgen-rayon for thread pool
    Ok(())
}
```

Threads require `SharedArrayBuffer` support, `COOP/COEP` headers, and `wasm-opt` compatibility flags.

## Debug Hooks

```rust
#[cfg(debug_assertions)]
#[wasm_bindgen(start)]
pub fn init_debug() {
    console_error_panic_hook::set_once();
}
```

```toml
[target.'cfg(debug_assertions)'.dependencies]
console_error_panic_hook = "0.1"
```

## JS Interop Shape

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(message: &str);
}

#[wasm_bindgen]
pub fn greet(name: &str) {
    log(&format!("hello, {name}"));
}
```

Prefer explicit bindings over passing untyped `JsValue` everywhere.

## Async Browser APIs

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;

#[wasm_bindgen]
pub async fn fetch_text(url: String) -> Result<String, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("missing window"))?;
    let response = JsFuture::from(window.fetch_with_str(&url)).await?;
    let response: web_sys::Response = response.dyn_into()?;
    let text = JsFuture::from(response.text()?).await?;
    text.as_string().ok_or_else(|| JsValue::from_str("response text was not a string"))
}
```

## Memory Management

Rust WASM memory (WASM linear memory) is separate from the JS heap:

- `wasm_bindgen` copies strings and arrays by default.
- Use `&[u8]` / `Vec<u8>` for zero-copy buffer sharing where possible.
- Large structures are more efficient as serialized bytes than as many small JS objects.
- Calling `free()` on JS objects (they're `JsValue` with Drop) is automatic when the Rust wrapper goes out of scope.

## Size Optimization

```bash
wasm-pack build --release --target web
wasm-opt -Oz -o pkg/optimized.wasm pkg/my_crate_bg.wasm
# Check size
ls -lh pkg/optimized.wasm
```

| Technique | Typical Savings |
|-----------|----------------|
| `opt-level = "z"` | 15-25% |
| LTO + 1 codegen unit | 10-20% |
| `wasm-opt -Oz` | 20-40% |
| Remove unused features | Variable |
| `wee_alloc` (legacy) | Smaller binary, slower alloc |

Modern Rust uses the default allocator; `wee_alloc` is rarely needed with LTO.

## WASM in Web Workers

```rust
// worker.rs — compiled separately as a dedicated wasm module
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn worker_entry() -> Result<(), JsValue> {
    // Worker main loop — receives messages via browser postMessage
    Ok(())
}
```

Workers cannot access the DOM (`web_sys::window()` returns `None`). Use `postMessage` for communication.

## WASI (Wasm on the Server)

```bash
rustup target add wasm32-wasip1
cargo build --target wasm32-wasip1 --release
# Run with wasmtime or wasmer
wasmtime run target/wasm32-wasip1/release/my-app.wasm
```

WASI gives you file I/O, clocks, and stdio — closer to a native binary than browser WASM.

## Testing WASM

```rust
// tests/web.rs
use wasm_bindgen_test::*;

#[wasm_bindgen_test]
fn test_filter_applies() {
    let filter = Filter::new(0.5);
    let mut pixels = vec![100u8, 150, 200, 255, 50, 75, 100, 255];
    filter.apply_rgba(&mut pixels);
    assert!(pixels[0] < 100); // darkened
}
```

```bash
wasm-pack test --firefox # or --chrome, --safari, --node
wasm-pack test --headless
```

## Anti-Patterns

```rust
// Bad: thousands of boundary crossings.
#[wasm_bindgen]
pub fn set_pixel(x: u32, y: u32, r: u8, g: u8, b: u8) { /* ... */ }

// Good: one call over a whole buffer.
#[wasm_bindgen]
pub fn apply_image_filter(rgba: &mut [u8]) { /* ... */ }
```

```rust
// Bad: unchecked browser globals panic in workers or tests.
let document = web_sys::window().unwrap().document().unwrap();

// Good: model environment availability as Result.
let document = web_sys::window()
    .and_then(|window| window.document())
    .ok_or_else(|| JsValue::from_str("document unavailable"))?;
```

```rust
// Bad: re-serializing unchanged data on every frame.
let js_val = serde_wasm_bindgen::to_value(&my_data).unwrap();

// Good: cache the serialized form if data hasn't changed.
static CACHED: OnceLock<JsValue> = OnceLock::new();
```

```rust
// Bad: large data copied across boundary repeatedly.
// The input array is cloned into WASM memory every frame.

// Good: allocate once in WASM, pass the pointer, reuse the buffer.
// See wasm-bindgen's support for manual memory management.
```

## Production Checklist

- Build release artifacts with `wasm-pack build --release` or equivalent.
- Run `wasm-opt -Oz` for size-sensitive web apps.
- Keep exported functions stable and documented like a public API.
- Avoid `serde_wasm_bindgen` on hot paths unless ergonomics beat copy cost.
- Test core logic natively with normal Rust tests; test bindings with `wasm-bindgen-test`.
- Verify target environment: browser main thread, worker, Node, edge runtime, or WASI.
- Set `panic = "abort"` in release profile.
- Measure bundle size in CI and set a budget.

## References

- [wasm-bindgen guide](https://rustwasm.github.io/docs/wasm-bindgen/)
- [Rust and WebAssembly book](https://rustwasm.github.io/docs/book/)
- [wasm-pack docs](https://rustwasm.github.io/docs/wasm-pack/)
- [web-sys crate docs](https://docs.rs/web-sys)
- [js-sys crate docs](https://docs.rs/js-sys)
- [wasm-opt tool](https://github.com/WebAssembly/binaryen)
