---
name: rust-dioxus
description: |
  Dioxus — cross-platform UI framework for Rust (web, desktop, mobile, SSR). Use this skill when:
  - User builds a web app, desktop app, or mobile app with Dioxus
  - User asks about Dioxus components, props, or RSX syntax
  - User works with Dioxus signals, state management, or reactivity
  - User uses use_signal, use_resource, use_effect, or use_memo
  - User implements routing with Dioxus Router
  - User asks about Dioxus full-stack (server functions, SSR)
  - User targets web (WASM), desktop (native), or mobile with Dioxus
  - User encounters Dioxus-specific compilation errors or rendering issues
---



## Quick Navigation

- [references/signals_reactivity.md](references/signals_reactivity.md)
- [references/fullstack_ssr.md](references/fullstack_ssr.md)

# Dioxus Cross-Platform UI Framework

A comprehensive guide to building beautiful, performant, and reactive user interfaces in Rust targeting WASM, Native Desktop, Mobile, and Server-Side Rendering (SSR).

## Quick Setup & Ecosystem

```bash
# Install the cargo helper utility
cargo install dioxus-cli

# Initialize a project template
dx new my-app --template fullstack
dx serve
```

```toml
# Cargo.toml
[dependencies]
dioxus = { version = "0.6", features = ["web"] } # or "desktop", "fullstack"
```

## Component Architecture & RSX Syntax

Dioxus components are regular functions annotated with `#[component]` returning an `Element`:

```rust
use dioxus::prelude::*;

#[component]
pub fn App() -> Element {
    rsx! {
        div {
            class: "app-container",
            Header { title: "Dioxus Dashboard" }
            MainContent {}
        }
    }
}

#[component]
fn Header(title: String) -> Element {
    rsx! {
        header { class: "nav-header", h1 { "{title}" } }
    }
}
```

### Key RSX Constraints
- **Attributes**: Write HTML/SVG attributes inline. Class names should use CSS names or utility framework classes (e.g. Tailwind).
- **Element keys**: Always specify a `key` attribute when rendering dynamic lists inside loops.

```rust
rsx! {
    ul {
        for user in users.read().iter() {
            li { key: "{user.id}", "{user.name}" }
        }
    }
}
```

## Reactivity & State Management (Dioxus 0.6)

Dioxus handles state tracking automatically via thread-safe Signals.

### Local Component State

```rust
#[component]
fn Counter() -> Element {
    let mut count = use_signal(|| 0);

    rsx! {
        button {
            onclick: move |_| count += 1, // Mutates value and triggers re-render
            "Count: {count}"
        }
    }
}
```

### Derived State: Memoization

Use `use_memo` to cache computations and prevent wasteful recalculations when dependencies haven't changed.

```rust
let value = use_signal(|| 10);
let doubled = use_memo(move || value() * 2);
```

### Global Shared Context

Provide and consume structured context globally down the virtual DOM tree without prop-drilling.

```rust
#[derive(Clone, Copy)]
struct AppState {
    authenticated: Signal<bool>,
}

// Parent Component
use_context_provider(|| AppState {
    authenticated: Signal::new(false),
});

// Child Component
let state = use_context::<AppState>();
```

## Async Resources & Server Integration

Use `use_resource` to run async fetching routines.

```rust
let posts = use_resource(move || async move {
    reqwest::get("https://api.example.com/posts")
        .await?
        .json::<Vec<Post>>()
        .await
});

match &*posts.read_unchecked() {
    None => rsx! { "Loading..." },
    Some(Err(e)) => rsx! { "Error: {e}" },
    Some(Ok(list)) => rsx! {
        for post in list {
            p { "{post.title}" }
        }
    }
}
```

### Server Functions (Full-Stack)

Define APIs that compile to RPC endpoints automatically.

```rust
#[server(GetDatabaseStats)]
pub async fn get_stats() -> Result<Stats, ServerFnError> {
    // This code executes purely on the backend server
    Ok(db::fetch_stats().await?)
}
```

## Desktop-Specific Invariants
Ensure you configure Window preferences elegantly on desktop launches:

```rust
fn main() {
    dioxus::LaunchBuilder::desktop()
        .with_cfg(
            dioxus::desktop::Config::default()
                .with_window(
                    dioxus::desktop::WindowBuilder::new()
                        .with_title("App")
                        .with_inner_size(dioxus::desktop::LogicalSize::new(1280, 720))
                )
        )
        .launch(App);
}
```

## Testing Components

Keep component logic in pure functions or hooks so behavior can be tested without a browser.

```rust
fn format_count(count: i32) -> String {
    match count {
        0 => "No items".to_string(),
        1 => "1 item".to_string(),
        n => format!("{n} items"),
    }
}

#[test]
fn formats_count() {
    assert_eq!(format_count(2), "2 items");
}
```

## Performance Guidelines

- Keep signals scoped to the smallest component that needs them.
- Avoid cloning large values into `rsx!`; pass IDs or `Arc` when appropriate.
- Use memoization for expensive derived state.
- Avoid blocking work in event handlers; use async tasks or server functions.
- Split large pages into components so rerenders stay localized.

```rust
let filtered = use_memo(move || {
    let query = query.read().to_lowercase();
    items.read()
        .iter()
        .filter(|item| item.name.to_lowercase().contains(&query))
        .cloned()
        .collect::<Vec<_>>()
});
```

## Error Handling

Represent loading, error, and success states explicitly. Do not unwrap in render paths.

```rust
#[derive(Clone)]
enum LoadState<T> {
    Idle,
    Loading,
    Loaded(T),
    Failed(String),
}
```

## Anti-Patterns

- Storing all app state in one global signal.
- Performing network requests directly in render logic.
- Blocking desktop event handlers with sync IO.
- Passing large owned props through many layers when context or IDs would be clearer.
- Mixing server-only APIs into client components without feature gates.

## Production Checklist

- Keep signals scoped near their consumers.
- Represent async resource states explicitly: loading, loaded, empty, failed.
- Avoid sync IO and CPU-heavy work in UI event handlers.
- Feature-gate server-only code and browser-only APIs.
- Split large views into components to limit rerender blast radius.
- Test core state transitions separately from rendering when possible.

## References

- Dioxus official documentation
- Dioxus examples repository
- Rust async and Tokio documentation for background task patterns
