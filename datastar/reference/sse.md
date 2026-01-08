# SSE Events Reference

Datastar uses Server-Sent Events (SSE) for backend-to-frontend communication. The backend streams events that update the DOM and signals.

## Content Type

```
Content-Type: text/event-stream
```

## Event Format

```
event: <event-type>
data: <key> <value>
data: <key> <value>

```

Note: Events are separated by double newlines.

## Event Types

### datastar-patch-elements

Updates DOM elements through morphing.

**Basic Usage:**
```
event: datastar-patch-elements
data: elements <div id="foo">Hello world!</div>

```

**Multi-line HTML:**
```
event: datastar-patch-elements
data: elements <div id="user-card">
data: elements   <h2>Alice</h2>
data: elements   <p>Software Engineer</p>
data: elements </div>

```

**Options:**

| Key | Default | Description |
|-----|---------|-------------|
| `selector` | - | CSS selector for target element |
| `mode` | `outer` | How to apply the patch |
| `namespace` | - | `svg` or `mathml` for XML content |
| `useViewTransition` | `false` | Enable view transitions |
| `elements` | - | The HTML content |

**Modes:**

| Mode | Description |
|------|-------------|
| `outer` | Replace entire element (morph outer HTML) |
| `inner` | Replace children only (morph inner HTML) |
| `replace` | Hard replace outer HTML (no morph) |
| `prepend` | Add as first child |
| `append` | Add as last child |
| `before` | Insert before element |
| `after` | Insert after element |
| `remove` | Remove the element |

**Examples:**

```
event: datastar-patch-elements
data: mode inner
data: selector #messages
data: elements <div class="message">New message!</div>

```

```
event: datastar-patch-elements
data: mode append
data: selector #list
data: elements <li>New item</li>

```

```
event: datastar-patch-elements
data: mode remove
data: selector #old-element

```

### datastar-patch-signals

Updates reactive signals (state) on the page.

**Basic Usage:**
```
event: datastar-patch-signals
data: signals {count: 42, name: "Alice"}

```

**Options:**

| Key | Default | Description |
|-----|---------|-------------|
| `signals` | - | JavaScript object with signal values |
| `onlyIfMissing` | `false` | Only set if signal doesn't exist |

**Remove signals** by setting to `null`:
```
event: datastar-patch-signals
data: signals {oldSignal: null}

```

**Conditional creation:**
```
event: datastar-patch-signals
data: onlyIfMissing true
data: signals {defaultValue: 0}

```

## Backend SDK Examples

### Go

```go
import "github.com/starfederation/datastar/sdk/go"

func handler(w http.ResponseWriter, r *http.Request) {
    // Read signals from request
    signals := datastar.ReadSignals(r)

    // Create SSE writer
    sse := datastar.NewSSE(w)

    // Patch elements
    sse.PatchElements("#result", "<div>Updated!</div>")

    // Patch with options
    sse.PatchElements("#list", "<li>Item</li>",
        datastar.WithMode("append"))

    // Patch signals
    sse.PatchSignals(map[string]any{
        "count": signals.Count + 1,
        "status": "complete",
    })
}
```

### Python

```python
from datastar import SSE, read_signals

def handler(request):
    signals = read_signals(request)

    sse = SSE()

    sse.patch_elements("#result", "<div>Updated!</div>")
    sse.patch_signals({"count": signals.count + 1})

    return sse.response()
```

### TypeScript

```typescript
import { SSE, readSignals } from '@starfederation/datastar';

export async function handler(req: Request): Promise<Response> {
    const signals = await readSignals(req);

    const sse = new SSE();

    sse.patchElements("#result", "<div>Updated!</div>");
    sse.patchSignals({ count: signals.count + 1 });

    return sse.response();
}
```

## Streaming Multiple Events

A single response can contain multiple events:

```
event: datastar-patch-signals
data: signals {status: "loading"}

event: datastar-patch-elements
data: selector #progress
data: elements <div class="progress" style="width: 50%"></div>

event: datastar-patch-signals
data: signals {status: "complete", progress: 100}

event: datastar-patch-elements
data: selector #result
data: elements <div id="result">Done!</div>

```

This enables progressive updates during long operations.

## Compression

SSE streams compress extremely well due to repetitive DOM patterns. Enable Brotli compression on your server for significant bandwidth savings (up to 200:1 ratios).

## Connection Management

- SSE connections are persistent
- Browser automatically reconnects on disconnect
- Use `data-on-interval` for polling patterns instead of keeping connections open for infrequent updates

## Error Handling

Send error messages by patching elements or signals:

```
event: datastar-patch-elements
data: selector #error
data: elements <div class="error">Something went wrong!</div>

event: datastar-patch-signals
data: signals {error: "Failed to save", isLoading: false}

```
