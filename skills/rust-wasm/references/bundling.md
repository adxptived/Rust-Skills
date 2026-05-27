# WASM Bundling and Deployment

Rust WASM output is only one part of the shipped artifact. Optimize the `.wasm`, generated JS glue, and the host bundler path together.

## wasm-pack

```bash
wasm-pack build --target web --release
wasm-pack build --target bundler --release
wasm-pack build --target nodejs --release
```

Choose the target based on the consumer:

| Target | Use for |
|--------|---------|
| `web` | Direct browser ESM imports |
| `bundler` | Vite, webpack, Rollup, Parcel |
| `nodejs` | Node-based tooling/tests |
| `no-modules` | Legacy script-tag loading |

## Trunk

```toml
# Cargo.toml
[dependencies]
yew = "0.21"
```

```html
<!-- index.html -->
<link data-trunk rel="rust" data-wasm-opt="z" />
```

```bash
trunk serve
trunk build --release
```

Use Trunk for Rust-first web apps. Use `wasm-pack` for libraries consumed by JS apps.

## Vite Integration

```ts
import init, { Filter } from './pkg/image_core.js';

await init();
const filter = new Filter(0.6);
filter.apply_rgba(pixelBuffer);
```

Initialize the generated module before calling exported Rust APIs. Keep initialization near app startup or worker startup.

## Size Optimization

```bash
wasm-pack build --target web --release
wasm-opt -Oz -o pkg/app_bg.opt.wasm pkg/app_bg.wasm
```

```toml
[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

## Workers

Move CPU-heavy WASM work off the browser main thread.

```ts
const worker = new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' });
worker.postMessage({ kind: 'process', buffer }, [buffer]);
```

Transfer `ArrayBuffer`s when ownership can move. Clone only when the caller must retain the original.

## Common Pitfalls

- Shipping debug `.wasm` artifacts with names and extra checks.
- Forgetting MIME type `application/wasm` on static hosts.
- Optimizing for size before checking parse/compile time in the browser.
- Using `target-cpu=native`; browsers run portable WASM, not host-native machine code.
- Assuming browser globals exist in Node, workers, or edge runtimes.

## Release Checklist

- Build with release profile and run `wasm-opt` when size matters.
- Confirm gzip/brotli compression on the static host.
- Check browser devtools for `.wasm` download size and compile time.
- Run smoke tests in target browsers and workers.
- Version JS glue and `.wasm` together; cache-bust both files.
