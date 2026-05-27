# Dioxus Reactivity: Architectural State Management

Dioxus 0.6 reactivity revolves around thread-safe `Signal<T>` cells. Readings are dynamically tracked during rendering, and updates trigger targeted virtual DOM reconciliation.

## 1. Local Signals & Closure Scope

Signals capture updates cleanly. When using signals inside callbacks, capture the signal itself by moving it instead of extracting its value beforehand.

```rust
use dioxus::prelude::*;

#[component]
fn ToggleButton() -> Element {
    let mut active = use_signal(|| false);

    rsx! {
        button {
            onclick: move |_| active.set(!active()),
            class: if active() { "active-btn" } else { "inactive-btn" },
            "Toggle"
        }
    }
}
```

## 2. Cloning vs Moving Signals

Signals implement `Copy` (they are `Copy + Clone` internally). This means you do not need to clone them into closures:

```rust
let mut count = use_signal(|| 0);

// Works: Signal is Copy
let handle = spawn(async move {
    loop {
        count += 1;
        sleep(1000).await;
    }
});
```

## 3. Memoization: use_memo

Use `use_memo` to avoid re-evaluating expensive derivation algorithms during component re-renders.

```rust
use dioxus::prelude::*;

#[component]
fn FilteredList(items: Signal<Vec<String>>, query: Signal<String>) -> Element {
    let filtered = use_memo(move || {
        let q = query().to_lowercase();
        items().iter()
            .filter(|item| item.to_lowercase().contains(&q))
            .cloned()
            .collect::<Vec<String>>()
    });

    rsx! {
        ul {
            for item in filtered() {
                li { "{item}" }
            }
        }
    }
}
```

## 4. Global State Architecture

Instead of passing states through multiple hierarchy layers (prop-drilling), provide state at the root level using `use_context_provider`.

```rust
use dioxus::prelude::*;

#[derive(Clone, Copy)]
struct AppConfig {
    api_url: Signal<String>,
}

#[component]
fn Root() -> Element {
    use_context_provider(|| AppConfig {
        api_url: Signal::new("https://api.example.com".to_string()),
    });
    
    rsx! { ChildComponent {} }
}

#[component]
fn ChildComponent() -> Element {
    let config = use_context::<AppConfig>();
    rsx! {
        span { "Endpoint: {config.api_url}" }
    }
}
```

## 5. Reactive Effects with use_effect

```rust
#[component]
fn Logger() -> Element {
    let count = use_signal(|| 0);

    // Runs after every render where `count` changed
    use_effect(move || {
        println!("Count is now: {}", count());
    });

    rsx! {
        button { onclick: move |_| count += 1, "Increment" }
    }
}
```

Use `use_effect` for side effects that should synchronize with state. Use `use_resource` for async data fetching.

## 6. use_resource for Async Data

```rust
#[component]
fn UserProfile(user_id: String) -> Element {
    let user = use_resource(move || async move {
        reqwest::get(format!("/api/users/{user_id}"))
            .await?
            .json::<User>()
            .await
    });

    match &*user.read_unchecked() {
        None => rsx! { "Loading..." },
        Some(Ok(u)) => rsx! { div { "Name: {u.name}" } },
        Some(Err(e)) => rsx! { "Error: {e}" },
    }
}
```

`use_resource` re-runs the async block whenever its captured signals change (in this case, `user_id`).

## 7. Signal Ownership Patterns

```rust
// Pattern A: signal as component prop
#[component]
fn Child(mut state: Signal<i32>) -> Element {
    rsx! {
        button { onclick: move |_| state += 1, "Inc" }
    }
}

// Pattern B: signal child of a struct
struct UiState {
    count: Signal<i32>,
    name: Signal<String>,
}

#[component]
fn App() -> Element {
    let ui = UiState {
        count: use_signal(|| 0),
        name: use_signal(|| "".to_string()),
    };
    rsx! {
        input { oninput: move |e| ui.name.set(e.value()), value: ui.name() }
        p { "Count: {ui.count}" }
    }
}
```

## 8. Resource Lifecycle Management

```rust
#[component]
fn ExpensiveResource(id: String) -> Element {
    let resource = use_resource(move || fetch_data(id.clone()));

    rsx! {
        match &*resource.read() {
            Some(Ok(data)) => render_data(data),
            Some(Err(e))   => rsx! { "Error: {e}" },
            None           => rsx! { "Loading..." },
        }
    }
}

// Resources are automatically cancelled when the component unmounts.
// Use AbortController or custom drop logic for long-running tasks.
```

## 9. Suspense Boundaries

Dioxus supports suspense-style fallback rendering:

```rust
#[component]
fn ProfilePage(user_id: String) -> Element {
    let fallback = rsx! { "Loading profile..." };

    rsx! {
        SuspenseBoundary { fallback,
            ProfileContent { user_id }
            SidebarContent { user_id }
        }
    }
}
```

Components inside the boundary suspend independently. The fallback shows until all suspended children resolve.

## 10. Avoiding Common Reactivity Pitfalls

```rust
// Bad: reading signal in component body, mutating in callback — causes unnecessary re-render
let count = use_signal(|| 0);
let doubled = count() * 2; // component captures this value once
rsx! {
    button { onclick: move |_| count += 1, "{doubled}" } // doubled never updates!
}

// Good: derive inside render, or use use_memo
let count = use_signal(|| 0);
let doubled = use_memo(move || count() * 2);
rsx! {
    button { onclick: move |_| count += 1, "{doubled()}" }
}
```

```rust
// Bad: creating signal inside loop — resets every render
for _ in 0..10 {
    let val = use_signal(|| 0); // NEW signal every render
}

// Good: use a single signal holding a collection
let vals = use_signal(|| vec![0; 10]);
```

## 11. Signal Read/Write Split

```rust
let (count, set_count) = use_signal(|| 0);

// `count` is read-only (impl Fn)
// `set_count` is write-only (impl FnMut)

// Pass read-only to display components:
rsx! { DisplayCount { count } }

// Pass write-only to input components:
rsx! { IncrementButton { set_count } }
```

This split prevents child components from accidentally reading and writing the same signal, which can cause subtle bugs.

## 12. Reactivity Summary

| Hook | Purpose | Re-runs when | Returns |
|------|---------|-------------|---------|
| `use_signal` | Local state | Signal is `.set()`/`.modify()` | `Signal<T>` (copy) |
| `use_memo` | Derived value | Dependencies change | `Signal<T>` (read-only) |
| `use_effect` | Side effects | Dependencies change, after render | `()` |
| `use_resource` | Async data | Captured signals change | `Resource<T>` |
| `use_context` | Shared state | Mount/unmount | `T` from provider |
| `use_ref` | Mutable ref | Never (no reactivity) | `&mut T` |
