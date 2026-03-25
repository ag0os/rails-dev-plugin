---
name: hotwire-patterns
description: Analyzes and recommends Hotwire patterns including Stimulus controllers, Turbo Frames, Turbo Streams, ActionCable broadcasts, and progressive enhancement for Rails frontends. Use when building interactive UI, partial page updates, real-time features, or form enhancements. NOT for REST API JSON responses, GraphQL, server-only background jobs, or model/database design.
allowed-tools: Read, Grep, Glob
---

# Hotwire Patterns for Rails

Analyze and recommend Stimulus + Turbo patterns for modern, interactive Rails applications.

## Quick Reference

| Component | Purpose | Use When |
|-----------|---------|----------|
| Stimulus | JS behaviors on HTML | Adding interactivity to server-rendered HTML |
| Turbo Drive | SPA-like navigation | Default for all links/forms (no config needed) |
| Turbo Frames | Partial page updates | Update a section without full reload |
| Turbo Streams | Multi-target updates | Update multiple DOM elements from one response |
| ActionCable + Streams | Real-time broadcasts | Push updates to all connected clients |

## Supporting Documentation

- [stimulus.md](stimulus.md) - Stimulus controller patterns and lifecycle
- [turbo.md](turbo.md) - Turbo Frames and Streams patterns

## Core Philosophy

**HTML over the wire**: Send HTML from the server, not JSON. JavaScript enhances server-rendered HTML.

1. **Progressive enhancement**: Works without JS, better with it
2. **Server-first**: Business logic stays on the server
3. **Minimal JavaScript**: Just enough JS to make HTML interactive
4. **No client-side state**: Server is the source of truth

## Stimulus Controller Template

```javascript
// app/javascript/controllers/dropdown_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu"]
  static classes = ["open"]
  static values = { open: { type: Boolean, default: false } }

  toggle() { this.openValue = !this.openValue }

  openValueChanged() {
    this.menuTarget.classList.toggle(this.openClass, this.openValue)
  }

  close(event) {
    if (!this.element.contains(event.target)) this.openValue = false
  }
}
```

```erb
<div data-controller="dropdown" data-dropdown-open-class="is-open"
     data-action="click@window->dropdown#close">
  <button data-action="dropdown#toggle">Menu</button>
  <div data-dropdown-target="menu">Content</div>
</div>
```

## Controller Communication

```javascript
// Outlets: direct reference to another controller
static outlets = ["search-results"]
filter() {
  if (this.hasSearchResultsOutlet) this.searchResultsOutlet.updateResults(query)
}

// Events: loosely coupled (dispatch + listen)
this.dispatch("filter", { detail: { query }, prefix: "search" })
// data-action="search:filter->results#handleFilter"
```

## Turbo Frames

```erb
<%# Scoped navigation - only replaces matching frame %>
<turbo-frame id="<%= dom_id(@post) %>">
  <%= render @post %>
  <%= link_to "Edit", edit_post_path(@post) %>
</turbo-frame>

<%# Lazy loading %>
<%= turbo_frame_tag "sidebar", src: sidebar_path, loading: :lazy do %>
  <p>Loading...</p>
<% end %>

<%# Break out of frame %>
<%= link_to "Full page", post_path(@post), data: { turbo_frame: "_top" } %>
```

## Turbo Streams

```erb
<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.prepend "posts", @post %>
<%= turbo_stream.update "posts-count", Post.count %>
<%= turbo_stream.replace "new-post-form" do %>
  <%= render "form", post: Post.new %>
<% end %>
```

## Broadcast Patterns (ActionCable)

```ruby
class Message < ApplicationRecord
  after_create_commit { broadcast_prepend_to "messages" }
  after_update_commit { broadcast_replace_to "messages" }
  after_destroy_commit { broadcast_remove_to "messages" }
  after_create_commit -> { broadcast_prepend_to(user, :notifications) }  # scoped
end
# View: <%= turbo_stream_from "messages" %>
```

## ActionCable + Stimulus Integration

```javascript
// app/javascript/controllers/chat_controller.js
import { Controller } from "@hotwired/stimulus"
import consumer from "../channels/consumer"

export default class extends Controller {
  static targets = ["messages", "input"]
  static values = { roomId: Number }

  connect() {
    this.subscription = consumer.subscriptions.create(
      { channel: "ChatChannel", room_id: this.roomIdValue },
      { received: (data) => this.messagesTarget.insertAdjacentHTML("beforeend", data.html) }
    )
  }
  disconnect() { this.subscription?.unsubscribe() }
  send(e) {
    e.preventDefault()
    if (this.inputTarget.value.trim()) {
      this.subscription.send({ message: this.inputTarget.value })
      this.inputTarget.value = ""
    }
  }
}
```

## Form Enhancements

```javascript
// Auto-submit with debounce
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }
  connect() { this.timeout = null }
  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => this.element.requestSubmit(), this.delayValue)
  }
}
// ERB: form_with data: { controller: "auto-submit", turbo_frame: "results" }
// Input: data: { action: "input->auto-submit#submit" }
```

## Lazy Loading

Use `IntersectionObserver` in a Stimulus controller to fetch content when elements scroll into view. Combine with `turbo-frame loading: :lazy` for server-rendered lazy frames. See [stimulus.md](stimulus.md) for full implementation.

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Client-side state management | Fights Hotwire's server-first model | Keep state on server, re-render HTML |
| Turbo Frame id mismatch | Silent failures, nothing updates | Match frame IDs exactly between pages |
| Missing `status: :unprocessable_entity` | Turbo won't render form errors | Always return 422 on validation failure |
| Fat Stimulus controllers (100+ lines) | Hard to maintain | Extract into multiple focused controllers |
| Inline JS instead of Stimulus | No lifecycle, no reuse | Use Stimulus controllers |
| Broadcasting without scoping | All users see all updates | Scope broadcasts to relevant streams |
| No `loading: :lazy` on hidden frames | Unnecessary requests on page load | Use lazy loading for below-fold content |

## Output Format

When analyzing or creating Hotwire components, provide:
1. **Stimulus controller** (JS) with targets, values, classes
2. **View partial** (ERB) with proper data attributes
3. **Controller action** with `turbo_stream` response format
4. **Model broadcasts** if real-time updates needed
5. **Cable subscription** if ActionCable integration required

## Error Handling

- Return `status: :unprocessable_entity` for form validation failures (Turbo requirement)
- Use `turbo_stream.replace` to re-render forms with error messages
- Handle Stimulus `connect`/`disconnect` lifecycle for cleanup (subscriptions, observers)
- Use `data-turbo-permanent` to preserve elements across navigation (flash, audio players)
- Set `data-turbo="false"` on links/forms that should not use Turbo (file downloads, external)
