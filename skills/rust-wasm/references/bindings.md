# WASM Bindings and Interop

Design bindings as a small, typed foreign-function interface. Every crossing between JavaScript and WebAssembly has overhead.

## Export Types Deliberately

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Parser {
    max_depth: usize,
}

#[wasm_bindgen]
impl Parser {
    #[wasm_bindgen(constructor)]
    pub fn new(max_depth: usize) -> Self {
        Self { max_depth }
    }

    pub fn parse(&self, input: &str) -> Result<JsValue, JsValue> {
        let ast = parse_with_limit(input, self.max_depth)
            .map_err(|err| JsValue::from_str(&err.to_string()))?;
        serde_wasm_bindgen::to_value(&ast).map_err(|err| JsValue::from_str(&err.to_string()))
    }
}
```

Use exported structs for stateful engines. Avoid reinitializing expensive Rust state from JS for every call.

## Choosing Data Shapes

| Data | Preferred boundary shape |
|------|---------------------------|
| Text | `&str` input, `String` output |
| Binary/image/audio | `&[u8]` or `&mut [u8]` |
| Small config | Struct with explicit fields |
| Complex JSON-like data | `serde_wasm_bindgen` |
| Hot path records | Columnar arrays or packed bytes |

## Errors

```rust
#[wasm_bindgen]
pub fn compile(pattern: &str) -> Result<RegexHandle, JsValue> {
    RegexHandle::new(pattern).map_err(|err| JsValue::from_str(&err.to_string()))
}
```

Return `Result<T, JsValue>` from exported fallible functions. Do not panic for normal validation failures.

## Async Interop

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::future_to_promise;
use js_sys::Promise;

#[wasm_bindgen]
pub fn run_job(input: String) -> Promise {
    future_to_promise(async move {
        let output = compute_async(input).await
            .map_err(|err| JsValue::from_str(&err.to_string()))?;
        Ok(JsValue::from_str(&output))
    })
}
```

Expose promises to JavaScript when the caller expects JS-native async behavior.

## Anti-Patterns

- Exporting dozens of fine-grained setters for hot loops.
- Passing `JsValue` through internal Rust layers instead of domain types.
- Using `unwrap()` for browser APIs that may be absent in workers.
- Serializing large buffers as JSON.
- Hiding global mutable state behind exported functions without lifecycle controls.

## Verification

- Unit-test pure Rust logic on the host target.
- Add `wasm-bindgen-test` for exported bindings.
- Exercise bindings from the same bundler/runtime users will use.
- Test both debug diagnostics and release artifact behavior.
