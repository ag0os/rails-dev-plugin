# Stimulus Controller Patterns

Detailed patterns for building Stimulus controllers in Rails applications.

## Controller Structure

### Basic Controller

```javascript
// app/javascript/controllers/example_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["output"]           // DOM elements to reference
  static classes = ["active", "hidden"] // CSS classes to toggle
  static values = {                     // Reactive data properties
    count: { type: Number, default: 0 },
    url: String
  }

  connect() {
    // Called when controller connects to DOM
  }

  disconnect() {
    // Called when controller disconnects
  }

  // Action methods
  increment() {
    this.countValue++
  }

  // Value change callbacks
  countValueChanged() {
    this.outputTarget.textContent = this.countValue
  }
}
```

### HTML Usage

```html
<div data-controller="example"
     data-example-count-value="5"
     data-example-url-value="/api/data"
     data-example-active-class="bg-blue-500"
     data-example-hidden-class="hidden">

  <span data-example-target="output">5</span>
  <button data-action="click->example#increment">+1</button>
</div>
```

## Common Patterns

### Dropdown Controller

```javascript
// app/javascript/controllers/dropdown_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu"]
  static classes = ["open"]
  static values = { open: { type: Boolean, default: false } }

  connect() {
    this.boundClose = this.closeOnClickOutside.bind(this)
  }

  toggle() {
    this.openValue = !this.openValue
  }

  close() {
    this.openValue = false
  }

  openValueChanged() {
    if (this.openValue) {
      this.menuTarget.classList.add(...this.openClasses)
      document.addEventListener('click', this.boundClose)
    } else {
      this.menuTarget.classList.remove(...this.openClasses)
      document.removeEventListener('click', this.boundClose)
    }
  }

  closeOnClickOutside(event) {
    if (!this.element.contains(event.target)) {
      this.close()
    }
  }

  disconnect() {
    document.removeEventListener('click', this.boundClose)
  }
}
```

### Modal Controller

```javascript
// app/javascript/controllers/modal_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["dialog"]
  static values = { open: Boolean }

  connect() {
    if (this.openValue) this.open()
  }

  open() {
    this.dialogTarget.showModal()
    document.body.classList.add('overflow-hidden')
  }

  close() {
    this.dialogTarget.close()
    document.body.classList.remove('overflow-hidden')
  }

  closeOnBackdrop(event) {
    if (event.target === this.dialogTarget) {
      this.close()
    }
  }

  closeOnEscape(event) {
    if (event.key === 'Escape') {
      this.close()
    }
  }
}
```

```erb
<div data-controller="modal">
  <button data-action="modal#open">Open Modal</button>

  <dialog data-modal-target="dialog"
          data-action="click->modal#closeOnBackdrop keydown.esc->modal#closeOnEscape">
    <div class="modal-content">
      <h2>Modal Title</h2>
      <p>Modal content here</p>
      <button data-action="modal#close">Close</button>
    </div>
  </dialog>
</div>
```

### Auto-Submit Form Controller

```javascript
// app/javascript/controllers/auto_submit_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }

  connect() {
    this.timeout = null
  }

  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }

  submitNow() {
    clearTimeout(this.timeout)
    this.element.requestSubmit()
  }
}
```

```erb
<%= form_with url: search_path, method: :get,
    data: { controller: "auto-submit", turbo_frame: "results" } do |f| %>
  <%= f.text_field :q, data: { action: "input->auto-submit#submit" } %>
  <%= f.select :category, categories,
      data: { action: "change->auto-submit#submitNow" } %>
<% end %>
```

### Form Validation Controller

```javascript
// app/javascript/controllers/form_validation_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["submit", "field"]

  connect() {
    this.validate()
  }

  validate() {
    const isValid = this.fieldTargets.every(field => field.checkValidity())
    this.submitTarget.disabled = !isValid
  }

  showError(event) {
    const field = event.target
    const errorElement = field.nextElementSibling

    if (!field.checkValidity() && errorElement?.classList.contains('error-message')) {
      errorElement.textContent = field.validationMessage
      errorElement.classList.remove('hidden')
    } else if (errorElement) {
      errorElement.classList.add('hidden')
    }
  }
}
```

### Clipboard Controller

```javascript
// app/javascript/controllers/clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source", "button"]
  static values = { successMessage: { type: String, default: "Copied!" } }

  async copy() {
    const text = this.sourceTarget.value || this.sourceTarget.textContent

    try {
      await navigator.clipboard.writeText(text)
      this.showSuccess()
    } catch (err) {
      console.error('Failed to copy:', err)
    }
  }

  showSuccess() {
    const originalText = this.buttonTarget.textContent
    this.buttonTarget.textContent = this.successMessageValue

    setTimeout(() => {
      this.buttonTarget.textContent = originalText
    }, 2000)
  }
}
```

## Controller Communication

### Using Outlets

```javascript
// app/javascript/controllers/filter_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["results"]

  filter(event) {
    const query = event.target.value
    if (this.hasResultsOutlet) {
      this.resultsOutlet.filterBy(query)
    }
  }
}

// app/javascript/controllers/results_controller.js
export default class extends Controller {
  static targets = ["item"]

  filterBy(query) {
    this.itemTargets.forEach(item => {
      const matches = item.textContent.toLowerCase().includes(query.toLowerCase())
      item.classList.toggle('hidden', !matches)
    })
  }
}
```

```erb
<div data-controller="filter" data-filter-results-outlet="#results-list">
  <input data-action="input->filter#filter">
</div>

<div id="results-list" data-controller="results">
  <div data-results-target="item">Item 1</div>
  <div data-results-target="item">Item 2</div>
</div>
```

### Using Custom Events

```javascript
// Dispatching events
this.dispatch("selected", {
  detail: { id: this.idValue },
  prefix: "item"  // Creates "item:selected" event
})

// Listening to events
<div data-action="item:selected->parent#handleSelection">
```

## Lifecycle Callbacks

```javascript
export default class extends Controller {
  initialize() {
    // Called once when controller is first instantiated
    // Good for one-time setup
  }

  connect() {
    // Called each time controller connects to DOM
    // Good for setting up event listeners, fetching data
  }

  disconnect() {
    // Called when controller disconnects from DOM
    // Good for cleanup: remove listeners, cancel requests
  }

  // Target callbacks
  outputTargetConnected(element) {
    // Called when a target element connects
  }

  outputTargetDisconnected(element) {
    // Called when a target element disconnects
  }

  // Value callbacks
  countValueChanged(value, previousValue) {
    // Called when value changes
  }
}
```

## Best Practices

### Do

- Keep controllers focused on single behaviors
- Use values for reactive state
- Use targets for DOM references (not `querySelector`)
- Clean up in `disconnect()`
- Use CSS classes values for styling
- Debounce expensive operations

### Don't

- Don't store state in instance variables (use values)
- Don't manipulate DOM outside your controller's element
- Don't make controllers too large (extract shared behavior)
- Don't rely on specific DOM structure (use targets)

## Directory Organization

```
app/javascript/controllers/
├── application.js
├── index.js
├── dropdown_controller.js
├── modal_controller.js
├── forms/
│   ├── validation_controller.js
│   └── auto_submit_controller.js
└── shared/
    ├── clipboard_controller.js
    └── loading_controller.js
```
