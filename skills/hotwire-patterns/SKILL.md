---
name: hotwire-patterns
description: Analyzes and recommends Hotwire patterns including Stimulus controllers, Turbo Frames, Turbo Streams, ActionCable broadcasts, and progressive enhancement for Rails frontends. Use when building interactive UI, partial page updates, real-time features, or form enhancements. NOT for REST API JSON responses, GraphQL, server-only background jobs, or model/database design.
allowed-tools: Read, Grep, Glob
---

# Hotwire Patterns for Rails

Analyze and recommend Stimulus + Turbo patterns for modern, interactive Rails applications.

## Quick Reference

| Component | Use When |
|-----------|----------|
| Stimulus | Adding JS behaviors to server-rendered HTML |
| Turbo Drive | Default for all links/forms (no config needed) |
| Turbo Frames | Update a section without full reload |
| Turbo Streams | Update multiple DOM elements from one response |
| ActionCable + Streams | Push real-time updates to connected clients |

## Supporting Documentation

- [stimulus.md](stimulus.md) - Stimulus integration patterns and production gotchas
- [turbo.md](turbo.md) - Turbo Frames, Streams, and version-aware patterns

## Core Philosophy

**HTML over the wire**: Send HTML from the server, not JSON. JavaScript enhances server-rendered HTML.

1. **Progressive enhancement**: Works without JS, better with it
2. **Server-first**: Business logic stays on the server
3. **Minimal JavaScript**: Just enough JS to make HTML interactive
4. **No client-side state**: Server is the source of truth

## Decision Guide: Frames vs Streams

| Scenario | Use | Why |
|----------|-----|-----|
| Edit-in-place | Frame | Scoped navigation, replaces itself |
| Form creates item + updates counter | Stream | Multiple targets from one response |
| Lazy sidebar | Frame with `loading: :lazy` | Deferred load, single target |
| Real-time chat | ActionCable + Stream | Push from server to all clients |
| Tabs/pagination | Frame | Scoped replacement |
| Flash + content update | Stream | Two targets: flash div + content |

## ActionCable + Stimulus Integration

This pattern is non-obvious -- it wires a Stimulus controller to an ActionCable subscription, giving you lifecycle-managed real-time behavior:

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

Key: `disconnect()` must unsubscribe to prevent leaked subscriptions during Turbo navigation.

## Critical Gotchas

### Missing 422 Status Code

Turbo will NOT render form error responses unless the server returns `status: :unprocessable_entity` (422). This is the #1 Hotwire debugging issue:

```ruby
# WRONG: Turbo ignores this response
format.html { render :new }

# RIGHT: Turbo processes the response
format.html { render :new, status: :unprocessable_entity }
```

### Frame ID Mismatch (Silent Failure)

If the response HTML does not contain a `<turbo-frame>` with a matching `id`, nothing happens -- no error, no update. Debug with:
- Browser console: look for "Response has no matching <turbo-frame id="...">" warning
- Verify `dom_id(@record)` produces the same ID on both pages
- Check that the edit/show view wraps content in the same frame tag

### Broadcasting Without Scoping

```ruby
# WRONG: every connected user sees every message
after_create_commit { broadcast_prepend_to "messages" }

# RIGHT: scope to the relevant stream
after_create_commit -> { broadcast_prepend_to(room, :messages) }
after_create_commit -> { broadcast_prepend_to(user, :notifications) }  # per-user
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Client-side state management | Fights Hotwire's server-first model | Keep state on server, re-render HTML |
| Fat Stimulus controllers (100+ lines) | Hard to maintain | Extract into multiple focused controllers |
| Broadcasting without scoping | All users see all updates | Scope broadcasts to relevant streams |
| No `loading: :lazy` on hidden frames | Unnecessary requests on page load | Use lazy loading for below-fold content |
| Streams when a Frame suffices | Overcomplicated | Use Frames for single-target scoped nav |

## Progressive Enhancement Checklist

Before shipping any Hotwire feature, verify:

- [ ] Core functionality works with Turbo Drive disabled (`data-turbo="false"`)
- [ ] Forms submit successfully without JS (standard HTML submission)
- [ ] Links navigate to full pages without Turbo Frames
- [ ] Stimulus controllers degrade gracefully (content visible without JS)
- [ ] `data-turbo-permanent` preserves media players, ActionCable connections across navigation
- [ ] `data-turbo="false"` on file downloads and external links

## Output Format

When analyzing or creating Hotwire components, provide:
1. **Stimulus controller** (JS) with targets, values, classes
2. **View partial** (ERB) with proper data attributes
3. **Controller action** with `turbo_stream` response format
4. **Model broadcasts** if real-time updates needed
5. **Cable subscription** if ActionCable integration required
