# Stimulus Patterns — Gotchas and Advanced Integration

Follow standard Stimulus controller conventions (targets, values, classes, lifecycle callbacks, actions). This file covers non-obvious patterns and production gotchas only.

## Auto-Save Controller (Production Pattern)

This pattern combines dirty-tracking, interval-based saves, and disconnect-save -- non-trivial to get right:

```javascript
// app/javascript/controllers/auto_save_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { interval: { type: Number, default: 3000 } }
  #timer
  #dirty = false

  disconnect() {
    this.submit()
  }

  change(event) {
    if (event.target.form === this.element && !this.#dirty) {
      this.#dirty = true
      this.#scheduleSave()
    }
  }

  async submit() {
    if (this.#dirty) {
      this.#resetTimer()
      this.#dirty = false
      await this.element.requestSubmit()
    }
  }

  #scheduleSave() {
    this.#timer = setTimeout(() => this.submit(), this.intervalValue)
  }

  #resetTimer() {
    clearTimeout(this.#timer)
  }
}
```

Key points:
- Uses private fields (`#dirty`, `#timer`) for encapsulation
- Saves on `disconnect()` (Turbo navigation away from page)
- `requestSubmit()` triggers Turbo form submission (not `submit()`)
- Combine with other controllers: `data-controller="autoresize auto-save"`

## Controller Communication: Outlets vs Events

**Use outlets** when controllers have a direct parent-child or sibling relationship and you need to call methods directly. **Use events** for loosely coupled controllers that may not know about each other.

```javascript
// Outlets: direct reference — requires data-filter-results-outlet="#results-list" in HTML
static outlets = ["results"]
filter() {
  if (this.hasResultsOutlet) this.resultsOutlet.filterBy(query)
}

// Events: loosely coupled — no HTML wiring needed between controllers
this.dispatch("filter", { detail: { query }, prefix: "search" })
// Listener: data-action="search:filter->results#handleFilter"
```

Gotcha: Outlet references break silently if the target element is removed from DOM (e.g., by a Turbo Stream update). Use `has*Outlet` guards.

## Lifecycle Gotchas with Turbo

Turbo navigation triggers `disconnect()` on the old page and `connect()` on the new page. This means:

1. **Always clean up in `disconnect()`**: event listeners, timers, subscriptions, observers
2. **Don't assume `connect()` runs once**: Turbo cache restoration calls `connect()` again with cached DOM
3. **Use `initialize()` for truly one-time setup** (runs once per controller instance creation)

```javascript
connect() {
  // May run multiple times due to Turbo cache
  this.observer = new IntersectionObserver(this.handleIntersect.bind(this))
  this.observer.observe(this.element)
}

disconnect() {
  // MUST clean up or you leak observers
  this.observer?.disconnect()
}
```

## Target Callbacks for Dynamic Content

When Turbo Streams add/remove elements, target callbacks fire automatically:

```javascript
// Fires when a new item is added to DOM (e.g., via Turbo Stream append)
itemTargetConnected(element) {
  this.updateCount()
  element.animate([{ opacity: 0 }, { opacity: 1 }], 300)
}

itemTargetDisconnected(element) {
  this.updateCount()
}
```

This is the recommended way to react to Turbo Stream DOM mutations inside a Stimulus controller.

## Directory Organization

```
app/javascript/
├── controllers/
│   ├── application.js
│   ├── index.js
│   ├── dropdown_controller.js
│   ├── forms/
│   │   ├── validation_controller.js
│   │   └── auto_submit_controller.js
│   └── shared/
│       └── clipboard_controller.js
└── helpers/
    └── timing_helpers.js
```

Namespace controllers in subdirectories. Stimulus auto-registers `forms/validation_controller.js` as `forms--validation`. Reference in HTML: `data-controller="forms--validation"`.
