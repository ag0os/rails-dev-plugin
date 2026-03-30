# Rails Views Patterns -- Detailed Reference

Follow standard Rails conventions for basic ERB, partials, layouts, `form_with`, `content_for`, and helper syntax. This file covers only non-obvious patterns and opinionated guidance.

## Collection Caching Strategies

### Russian-Doll Caching

Nest caches so inner updates only bust the inner fragment. The outer model must `touch` on inner changes:

```ruby
class Product < ApplicationRecord
  belongs_to :category, touch: true
end
```

```erb
<% cache @category do %>
  <h2><%= @category.name %></h2>
  <%= render partial: "products/product", collection: @category.products, cached: true %>
<% end %>
```

Key points:
- `cached: true` on collection render uses `read_multi` to batch cache reads
- The `touch: true` association ensures the outer cache busts when any inner record changes
- Use `cache_if(condition, record)` when admins should see uncached content

### Conditional Caching

```erb
<% cache_if(!current_user&.admin?, @product) do %>
  <%= render @product %>
<% end %>
```

## ViewComponent Patterns (Service-Oriented)

**Profile:** ViewComponents are a service-oriented pattern. **Omakase** projects prefer helpers, partials, and presenter POROs. Use ViewComponents only when the `view_component` gem is already in the project.

```ruby
# app/components/alert_component.rb
class AlertComponent < ViewComponent::Base
  ALLOWED_TYPES = %i[info success warning error].freeze

  def initialize(type: :info, dismissible: false)
    @type = ALLOWED_TYPES.include?(type) ? type : :info
    @dismissible = dismissible
  end
end
```

```erb
<%# app/components/alert_component.html.erb %>
<div class="alert alert--<%= @type %>" role="alert"
     <% if @dismissible %>data-controller="dismissible"<% end %>>
  <%= content %>
  <% if @dismissible %>
    <button data-action="dismissible#dismiss" aria-label="Dismiss">&times;</button>
  <% end %>
</div>
```

## Form Object Pattern

Use when a form doesn't map 1:1 to a model, or spans multiple models:

```ruby
# app/forms/registration_form.rb
class RegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :email, :string
  attribute :password, :string
  attribute :terms_accepted, :boolean

  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, length: { minimum: 8 }
  validates :terms_accepted, acceptance: true

  def save
    return false unless valid?
    User.create!(email: email, password: password)
  end
end
```

This works with `form_with model: RegistrationForm.new` like any ActiveRecord model.

## Accessibility Patterns

### Accessible Error States

Link errors to fields via `aria-describedby` and announce with `role="alert"`:

```erb
<%= f.email_field :email, aria: { invalid: @user.errors[:email].any?, describedby: "email-error" } %>
<% if @user.errors[:email].any? %>
  <span id="email-error" role="alert"><%= @user.errors[:email].first %></span>
<% end %>
```

### Flash Messages as Live Regions

```erb
<div aria-live="polite" aria-atomic="true">
  <% flash.each do |type, message| %>
    <div class="flash flash--<%= type %>" role="status"><%= message %></div>
  <% end %>
</div>
```

### Skip Navigation

Must be the first focusable element inside `<body>`:

```erb
<a href="#main-content" class="sr-only sr-only--focusable">Skip to main content</a>
```

## Performance Decision Table

| Technique | When to Use |
|-----------|------------|
| `render collection:` | Rendering 3+ items of the same partial |
| `cached: true` on collection | Collection items rarely change |
| Russian-doll caching | Nested parent/child displays with `touch: true` |
| `cache_if(condition)` | Skip caching for admins or preview modes |
| `loading="lazy"` on images | Below-the-fold images |
| Turbo Frames | Defer loading non-critical page sections |
