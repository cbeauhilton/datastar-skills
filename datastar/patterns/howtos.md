# Datastar How-To Patterns

Common implementation patterns for Datastar applications.

## Form Handling

### Basic Form Submission

```html
<form data-signals="{email: '', password: ''}"
      data-on:submit.prevent="@post('/login')">
  <input type="email" data-bind:value="email" required>
  <input type="password" data-bind:value="password" required>
  <button type="submit" data-indicator="#submitting">Login</button>
  <span id="submitting" style="display:none">Logging in...</span>
</form>
<div id="login-result"></div>
```

Backend responds with:
```
event: datastar-patch-elements
data: selector #login-result
data: elements <div class="success">Welcome back!</div>
```

### Inline Validation

```html
<form data-signals="{username: ''}">
  <input type="text"
         data-bind:value="username"
         data-on:blur="@post('/validate/username')">
  <span id="username-error"></span>
</form>
```

### File Upload

```html
<form data-signals="{_file: null}">
  <input type="file"
         data-on:change="_file = evt.target.files[0]">
  <button data-on:click="@post('/upload', {body: _file})"
          data-indicator="#uploading">
    Upload
  </button>
  <div id="uploading" style="display:none">Uploading...</div>
</form>
```

## Lists and Tables

### Click to Edit Row

```html
<table>
  <tr id="user-1">
    <td>Alice</td>
    <td><button data-on:click="@get('/users/1/edit')">Edit</button></td>
  </tr>
</table>
```

Backend replaces row with edit form:
```
event: datastar-patch-elements
data: selector #user-1
data: elements <tr id="user-1">
data: elements   <td><input data-bind:value="name" value="Alice"></td>
data: elements   <td>
data: elements     <button data-on:click="@post('/users/1')">Save</button>
data: elements     <button data-on:click="@get('/users/1')">Cancel</button>
data: elements   </td>
data: elements </tr>
```

### Delete Row with Confirmation

```html
<tr id="item-42">
  <td>Item Name</td>
  <td>
    <button data-on:click="@delete('/items/42')"
            data-confirm="Are you sure?">
      Delete
    </button>
  </td>
</tr>
```

### Infinite Scroll / Load More

```html
<div id="items">
  <!-- Existing items -->
</div>
<div data-on-intersect.once="@get('/items?page=2')"
     id="load-more">
  Loading more...
</div>
```

Backend appends new items and updates trigger:
```
event: datastar-patch-elements
data: mode append
data: selector #items
data: elements <div class="item">New Item 1</div>
data: elements <div class="item">New Item 2</div>

event: datastar-patch-elements
data: selector #load-more
data: elements <div data-on-intersect.once="@get('/items?page=3')" id="load-more">
data: elements   Loading more...
data: elements </div>
```

## Real-Time Updates

### Polling for Status

```html
<div id="job-status"
     data-on-interval="2000; @get('/jobs/123/status')">
  Checking status...
</div>
```

### Live Notifications

Keep a persistent SSE connection for push updates:

```html
<div data-init="@get('/notifications/stream')">
  <ul id="notifications"></ul>
</div>
```

Backend sends events as they occur:
```
event: datastar-patch-elements
data: mode prepend
data: selector #notifications
data: elements <li class="notification new">New message from Alice</li>

```

## Search and Filtering

### Debounced Search

```html
<div data-signals="{query: ''}">
  <input type="search"
         data-bind:value="query"
         data-on:input.debounce_300ms="@get('/search')"
         placeholder="Search...">
  <div id="results"></div>
</div>
```

### Filter Controls

```html
<div data-signals="{category: 'all', sort: 'newest'}">
  <select data-bind:value="category"
          data-on:change="@get('/products')">
    <option value="all">All Categories</option>
    <option value="electronics">Electronics</option>
    <option value="clothing">Clothing</option>
  </select>

  <select data-bind:value="sort"
          data-on:change="@get('/products')">
    <option value="newest">Newest</option>
    <option value="price-low">Price: Low to High</option>
    <option value="price-high">Price: High to Low</option>
  </select>

  <div id="product-list"></div>
</div>
```

## UI Interactions

### Toggle Menu (Local State)

```html
<div data-signals="{_menuOpen: false}">
  <button data-on:click="_menuOpen = !_menuOpen">
    Menu
  </button>
  <nav data-show="_menuOpen">
    <a href="/home">Home</a>
    <a href="/about">About</a>
    <a href="/contact">Contact</a>
  </nav>
</div>
```

### Modal Dialog

```html
<div data-signals="{_modalOpen: false, _modalContent: ''}">
  <button data-on:click="_modalOpen = true; @get('/modal/confirm-delete')">
    Delete Account
  </button>

  <div data-show="_modalOpen" class="modal-backdrop">
    <div class="modal" id="modal-content">
      <!-- Content loaded here -->
    </div>
  </div>
</div>
```

### Tabs

```html
<div data-signals="{_activeTab: 'overview'}">
  <div class="tabs">
    <button data-on:click="_activeTab = 'overview'"
            data-class="{'active': _activeTab === 'overview'}">
      Overview
    </button>
    <button data-on:click="_activeTab = 'details'; @get('/tabs/details')"
            data-class="{'active': _activeTab === 'details'}">
      Details
    </button>
    <button data-on:click="_activeTab = 'reviews'; @get('/tabs/reviews')"
            data-class="{'active': _activeTab === 'reviews'}">
      Reviews
    </button>
  </div>

  <div data-show="_activeTab === 'overview'" id="tab-overview">
    <!-- Static overview content -->
  </div>
  <div data-show="_activeTab === 'details'" id="tab-details">
    <!-- Loaded on demand -->
  </div>
  <div data-show="_activeTab === 'reviews'" id="tab-reviews">
    <!-- Loaded on demand -->
  </div>
</div>
```

## Keyboard Shortcuts

### Global Key Bindings

```html
<div data-on:keydown__window="
  if (evt.key === 'Escape') _modalOpen = false;
  if (evt.key === '/' && evt.ctrlKey) $refs.search.focus();
">
  <input data-ref="search" type="search">
</div>
```

### Form Navigation

```html
<form data-on:keydown="
  if (evt.key === 'Enter' && evt.ctrlKey) @post('/submit');
">
  <textarea data-bind:value="content"></textarea>
  <small>Ctrl+Enter to submit</small>
</form>
```

## Progress and Loading

### Progress Bar

```html
<div data-signals="{progress: 0}">
  <button data-on:click="@post('/start-job')"
          data-indicator="#processing">
    Start Processing
  </button>
  <div id="processing" style="display:none">
    <div class="progress-bar">
      <div class="progress-fill"
           data-style:width="`${progress}%`"></div>
    </div>
    <span data-text="`${progress}% complete`"></span>
  </div>
</div>
```

Backend streams progress updates:
```
event: datastar-patch-signals
data: signals {progress: 25}

event: datastar-patch-signals
data: signals {progress: 50}

event: datastar-patch-signals
data: signals {progress: 75}

event: datastar-patch-signals
data: signals {progress: 100}

event: datastar-patch-elements
data: selector #processing
data: elements <div id="processing"><p>Complete!</p></div>
```

## Preventing SSE Connection Timeouts

For long-lived connections, send periodic keep-alive events:

```go
// Backend sends empty comment every 30 seconds
ticker := time.NewTicker(30 * time.Second)
for {
    select {
    case <-ticker.C:
        fmt.Fprintf(w, ": keepalive\n\n")
        flusher.Flush()
    case event := <-events:
        // Send actual event
    }
}
```

## Backend Redirect

Redirect from the backend by patching a signal that triggers navigation:

```html
<div data-signals="{_redirect: ''}"
     data-effect="if (_redirect) window.location.href = _redirect">
</div>
```

Or use the standard approach - send a redirect header and let the browser handle it.
