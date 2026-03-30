# Rails Views Patterns -- Detailed Reference

## Fragment Caching Patterns

### Basic Fragment Cache

```erb
<% cache @product do %>
  <%= render @product %>
<% end %>
```

The cache key is auto-generated from `@product.cache_key_with_version`, so it busts when the record updates.

### Russian-Doll Caching

Nest caches so inner updates only bust the inner fragment:

```erb
<% cache @category do %>
  <h2><%= @category.name %></h2>
  <% @category.products.each do |product| %>
    <% cache product do %>
      <%= render product %>
    <% end %>
  <% end %>
<% end %>
```

Ensure the outer model touches on inner changes:

```ruby
class Product < ApplicationRecord
  belongs_to :category, touch: true
end
```

### Collection Caching

```erb
<%= render partial: "products/product", collection: @products, cached: true %>
```

Rails batches cache reads via `read_multi`, dramatically reducing cache round-trips.

### Cache with Expiry

```erb
<% cache @dashboard, expires_in: 15.minutes do %>
  <%= render "dashboard/stats" %>
<% end %>
```

## Collection Rendering Patterns

### Basic Collection

```erb
<%# Automatically renders _product.html.erb for each item %>
<%= render @products %>

<%# Explicit form with spacer %>
<%= render partial: "product", collection: @products,
           spacer_template: "product_divider" %>
```

### Empty Collection Fallback

```erb
<%= render(@products) || render("products/empty_state") %>
```

### Collection Counter

Rails provides a `product_counter` local (0-indexed) automatically:

```erb
<%# app/views/products/_product.html.erb %>
<div class="product" id="product-<%= product_counter %>">
  <span class="rank"><%= product_counter + 1 %></span>
  <h3><%= product.name %></h3>
</div>
```

## ViewComponent Patterns (Service-Oriented)

**Profile:** ViewComponents are a service-oriented pattern. **Omakase** projects prefer helpers, partials, and presenter POROs instead. Use ViewComponents when the `view_component` gem is already in the project.

When using the `view_component` gem:

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

```erb
<%# Usage in any view %>
<%= render AlertComponent.new(type: :success, dismissible: true) do %>
  Record saved successfully.
<% end %>
```

## Form Patterns

### Nested Attributes

```erb
<%= form_with model: @project do |f| %>
  <%= f.text_field :name %>

  <%= f.fields_for :tasks do |task_form| %>
    <%= task_form.text_field :title %>
    <%= task_form.check_box :_destroy, label: "Remove" %>
  <% end %>

  <%= f.submit %>
<% end %>
```

### Rich Form Controls

```erb
<%= form_with model: @post do |f| %>
  <%# Select with grouped options %>
  <%= f.grouped_collection_select :category_id,
        Department.all, :categories, :name, :id, :name %>

  <%# Date select %>
  <%= f.date_field :published_on %>

  <%# File upload %>
  <%= f.file_field :cover_image, accept: "image/*",
        direct_upload: true %>
<% end %>
```

### Form Object Pattern

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

## Helper Patterns

### Presentational Helpers

```ruby
# app/helpers/status_helper.rb
module StatusHelper
  STATUS_CLASSES = {
    "active"   => "badge badge--green",
    "pending"  => "badge badge--yellow",
    "archived" => "badge badge--gray"
  }.freeze

  def status_badge(status)
    css = STATUS_CLASSES.fetch(status, "badge")
    tag.span(status.humanize, class: css)
  end
end
```

### Conditional Rendering

```ruby
# app/helpers/application_helper.rb
def render_if(condition, partial, **locals)
  render(partial, **locals) if condition
end
```

## Layout Patterns

### Multiple Layouts

```ruby
class AdminController < ApplicationController
  layout "admin"
end
```

### Conditional Content Regions

```erb
<%# app/views/layouts/application.html.erb %>
<html>
<head>
  <%= yield :head %>
</head>
<body>
  <% if content_for?(:sidebar) %>
    <aside class="sidebar"><%= yield :sidebar %></aside>
  <% end %>
  <main class="<%= content_for?(:sidebar) ? 'with-sidebar' : 'full-width' %>">
    <%= yield %>
  </main>
</body>
</html>
```

```erb
<%# In a specific view %>
<% content_for :sidebar do %>
  <%= render "shared/filters" %>
<% end %>
```

## Accessibility Patterns

### Accessible Forms

```erb
<%= form_with model: @user do |f| %>
  <div class="field" role="group" aria-labelledby="name-group">
    <span id="name-group" class="sr-only">Name fields</span>
    <%= f.label :first_name %>
    <%= f.text_field :first_name, required: true, aria: { required: true } %>

    <%= f.label :last_name %>
    <%= f.text_field :last_name, required: true %>
  </div>

  <% if @user.errors[:email].any? %>
    <%= f.label :email %>
    <%= f.email_field :email, aria: { invalid: true, describedby: "email-error" } %>
    <span id="email-error" role="alert"><%= @user.errors[:email].first %></span>
  <% end %>
<% end %>
```

### Skip Navigation

```erb
<%# Very first element inside <body> %>
<a href="#main-content" class="sr-only sr-only--focusable">
  Skip to main content
</a>
```

### ARIA Live Regions for Flash Messages

```erb
<div aria-live="polite" aria-atomic="true">
  <% flash.each do |type, message| %>
    <div class="flash flash--<%= type %>" role="status">
      <%= message %>
    </div>
  <% end %>
</div>
```

## Performance Tips

| Technique | When to Use |
|-----------|------------|
| `render collection:` | Rendering 5+ items of the same partial |
| `cached: true` on collection | Collection items rarely change |
| Russian-doll caching | Nested parent/child displays |
| `cache_if(condition)` | Conditional caching for admin vs public |
| Lazy-load images | Below-the-fold images (`loading="lazy"`) |
| Turbo Frames | Defer loading non-critical page sections |
| `content_for` | Inject page-specific JS/CSS into layout |
