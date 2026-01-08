---
name: datastar
description: Hypermedia framework for building reactive web applications with backend-driven UI. Use this skill for Datastar development patterns, SSE streaming, signal management, DOM morphing, and the "Datastar way" philosophy. Covers data-* attributes, backend integration, and real-time collaborative app patterns.
version: 1.0.0
---

# Datastar

Datastar is a lightweight (~11KB) hypermedia framework for building reactive web applications. It enables server-side rendering with frontend framework capabilities by accepting HTML and SSE responses. The backend drives the frontend - this is the core philosophy.

## The Tao of Datastar (Philosophy)

The "Datastar way" is a set of opinions from the core team on building maintainable, scalable, high-performance web apps.

### Core Principles

1. **Backend is Source of Truth**: Most application state should reside on the server. The frontend is exposed to users and cannot be trusted.

2. **Sensible Defaults**: Resist the urge to customize before questioning whether changes are truly necessary.

3. **DOM Patching Over Fine-Grained Updates**: The backend drives frontend changes by patching HTML elements and signals. Send large DOM chunks ("fat morph") efficiently.

4. **Morphing as Core Strategy**: Only modified parts of the DOM are updated, preserving state and improving performance.

5. **Restrained Signal Usage**: Signals should serve specific purposes:
   - User interactions (toggling visibility)
   - Binding form inputs to backend requests
   - Overusing signals indicates inappropriate frontend state management

### Anti-Patterns to AVOID

- **Optimistic Updates**: Don't update UI speculatively. Use loading indicators and confirm outcomes only after backend verification.
- **Assuming Fresh Frontend State**: Don't trust frontend cache. Fetch current state from backend.
- **Custom History Management**: Don't manually manipulate browser history. Use standard navigation.

## Architecture: Event Streams All The Way Down

Datastar leverages Server-Sent Events (SSE) as its foundation. The server pushes data through a persistent connection.

**Key insight**: Datastar extends `text/event-stream` beyond SSE's GET-only limitation. HTML fragments wrapped in this protocol support POST, PUT, PATCH, DELETE operations.

### Request/Response Flow

1. User triggers action (click, form submission)
2. Frontend serializes ALL signals (except `_`-prefixed) and sends request
3. Backend reads signals and processes business logic
4. Server streams SSE events back to browser
5. Frontend receives events and updates DOM/signals reactively

## SSE Event Types

### `datastar-patch-elements`

Modifies DOM elements through morphing.

```
event: datastar-patch-elements
data: elements <div id="foo">Hello world!</div>
```

**Options:**
- `selector` - CSS selector for target element
- `mode` - `outer` (default), `inner`, `replace`, `prepend`, `append`, `before`, `after`, `remove`
- `useViewTransition` - Enable view transitions

### `datastar-patch-signals`

Updates reactive state on the page.

```
event: datastar-patch-signals
data: signals {foo: 1, bar: 2}
```

Remove signals by setting to `null`.

## Core Attributes

### Event Handling
- `data-on:click` - Handle click events
- `data-on:submit` - Handle form submissions
- `data-on:keydown__window` - Global key events (with `__window` modifier)
- `data-on-intersect` - Visibility triggers
- `data-on-interval` - Periodic triggers

### State Management
- `data-signals` - Define local signals: `data-signals="{count: 0}"`
- `data-bind` - Two-way binding: `data-bind:value="email"`
- `data-computed` - Derived values
- `data-init` - Initialize on load

### DOM Updates
- `data-text` - Set text content: `data-text="$count"`
- `data-show` - Conditional visibility
- `data-class` - Dynamic classes
- `data-attr` - Dynamic attributes
- `data-ref` - Element references

### Morphing Control
- `data-ignore` - Skip this element during morph
- `data-ignore-morph` - Preserve element across morphs
- `data-preserve-attr` - Keep specific attributes

### Backend Actions
- `@get('/path')` - GET request
- `@post('/path')` - POST request
- `@put('/path')` - PUT request
- `@patch('/path')` - PATCH request
- `@delete('/path')` - DELETE request

### Utility Actions
- `@peek()` - Access signals without subscribing
- `@setAll()` - Set all matching signals
- `@toggleAll()` - Toggle boolean signals

## Signal Naming Convention

- Regular signals: sent with every request
- `_`-prefixed signals: local only, NOT sent to backend

```html
<div data-signals="{username: '', _isMenuOpen: false}">
  <!-- username goes to backend, _isMenuOpen stays local -->
</div>
```

## Backend SDK Pattern

All SDKs provide helpers for reading signals and streaming responses:

```go
// Go example
func handler(w http.ResponseWriter, r *http.Request) {
    signals := datastar.ReadSignals(r)

    sse := datastar.NewSSE(w)
    sse.PatchElements("#result", "<div>Updated!</div>")
    sse.PatchSignals(map[string]any{"count": signals.Count + 1})
}
```

Available SDKs: Go, Python, TypeScript, PHP, Ruby, Rust, Java, Kotlin, Scala, C#, Clojure

## Common Patterns

### Loading Indicators

```html
<button data-on:click="@post('/submit')"
        data-indicator="#loading">
  Submit
</button>
<span id="loading" style="display:none">Loading...</span>
```

### Form Binding

```html
<form data-signals="{email: '', password: ''}">
  <input type="email" data-bind:value="email">
  <input type="password" data-bind:value="password">
  <button data-on:click="@post('/login')">Login</button>
</form>
```

### Conditional Rendering

```html
<div data-signals="{_showDetails: false}">
  <button data-on:click="_showDetails = !_showDetails">Toggle</button>
  <div data-show="_showDetails">
    Details here...
  </div>
</div>
```

### Polling

```html
<div data-on-interval="1000; @get('/status')">
  <!-- Refreshes every second -->
</div>
```

### Lazy Loading

```html
<div data-on-intersect="@get('/load-more')">
  Loading...
</div>
```

## CQRS Pattern

Segregate commands (writes) from queries (reads):
- **Long-lived read connections**: Real-time updates via SSE
- **Short-lived write requests**: POST/PUT/DELETE

This enables real-time collaboration patterns.

## Performance Tips

1. **Use Compression**: Brotli on SSE streams can achieve 200:1 ratios due to repetitive DOM data
2. **Fat Morph**: Don't fear sending large HTML chunks - morphing is efficient
3. **Debounce Input**: Use `data-on:input.debounce_500ms` for search fields

## When to Use This Skill

- Building reactive web UIs without heavy JS frameworks
- Implementing real-time features (chat, notifications, live updates)
- Converting traditional server-rendered apps to be more interactive
- Questions about Datastar attributes, patterns, or philosophy
- Debugging SSE connections or signal behavior
- Choosing between client-side and server-side state

## Skill Contents

- `patterns/` - Common implementation patterns and anti-patterns
- `reference/` - Attribute and action reference documentation

## Links

- [Official Documentation](https://data-star.dev)
- [GitHub Repository](https://github.com/starfederation/datastar)
- [Discord Community](https://discord.gg/datastar)
- [Examples](https://data-star.dev/examples)
