---
name: rails-views-patterns
description: Analyzes Rails view templates, partials, layouts, helpers, and form patterns for best practices. Use when reviewing ERB templates, improving view performance with fragment caching, fixing form helpers, organizing partials, adding accessibility attributes, or evaluating collection rendering. NOT for Stimulus/Turbo logic (use hotwire-patterns), controller concerns, or API-only responses.
allowed-tools: Read, Grep, Glob
---

# Rails Views Patterns

Analyze and recommend best practices for Rails view layer code including ERB templates, partials, helpers, forms, caching, and accessibility.

See [patterns.md](patterns.md) for detailed code examples.

## Quick Reference

| Area | Key Rule | Anti-Pattern |
|------|----------|--------------|
| Partials | Extract reusable components | Duplicated HTML across views |
| Helpers | Format data for display | Business logic in helpers |
| Forms | Always use `form_with` | Raw `<form>` tags |
| Caching | Fragment cache expensive renders | No caching on collection renders |
| Collections | Use `render collection:` | Manual loops with `each` |
| Accessibility | Semantic HTML + ARIA | `<div>` soup without roles |
| Layouts | Use `yield :section` for regions | Inline styles/scripts in templates |

## Core Principles

1. **Views are for presentation only** -- no business logic, no queries, no mutations
2. **Use Rails helpers** -- `link_to`, `image_tag`, `form_with`, `content_for`
3. **Partials for reuse** -- extract shared markup into `_partial.html.erb`
4. **Semantic HTML5** -- `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`
5. **Cache aggressively** -- fragment caching for expensive or repeated renders

## Key Patterns

### Layouts and Content Regions

```erb
<%# app/views/layouts/application.html.erb %>
<%= content_for?(:head) ? yield(:head) : "" %>
<%= render "shared/header" %>
<main><%= yield %></main>
<%= render "shared/footer" %>
```

### View Helpers

```ruby
# app/helpers/application_helper.rb
def format_date(date)
  date.strftime("%B %d, %Y") if date.present?
end

def active_link_to(name, path, **options)
  options[:class] = [options[:class], ("active" if current_page?(path))].compact.join(" ")
  link_to name, path, options
end
```

### Forms with `form_with`

```erb
<%= form_with model: @user do |f| %>
  <%= f.label :email %>
  <%= f.email_field :email, class: "form-control", aria: { describedby: "email-help" } %>
  <span id="email-help" class="help-text">We'll never share your email.</span>
  <%= f.submit class: "btn btn-primary" %>
<% end %>
```

### Collection Rendering with Caching

```erb
<%# Preferred: automatic counter/spacer support + caching %>
<%= render partial: "product", collection: @products, cached: true %>
```

### Fragment Caching

```erb
<% cache @product do %>
  <div class="product-card">
    <h2><%= @product.name %></h2>
    <p><%= @product.description %></p>
  </div>
<% end %>
```

## Presenter Objects (Omakase)

**Omakase profile:** When view logic gets complex, use a plain Ruby object as a presenter — not a ViewComponent.

```ruby
# app/models/column.rb (or app/presenters/column.rb)
class Column
  attr_reader :events, :date

  def initialize(events:, date:)
    @events = events
    @date = date
  end

  def grouped_events
    events.group_by(&:category).sort_by { |cat, _| cat.position }
  end

  def empty? = events.none?
  def title  = date.strftime("%A, %B %d")
end
```

```erb
<%# In view — clean API, no logic %>
<% @columns.each do |column| %>
  <div class="column">
    <h3><%= column.title %></h3>
    <% column.grouped_events.each do |category, events| %>
      <%= render partial: "event", collection: events %>
    <% end %>
  </div>
<% end %>
```

**Service-oriented profile:** ViewComponents are also fine — see [patterns.md](patterns.md) for ViewComponent patterns.

## Anti-Patterns

| Bad | Good | Why |
|-----|------|-----|
| `<% @users = User.all %>` in view | Pass `@users` from controller | Views must not query |
| `<% if user.admin? && user.active? && ... %>` | Extract to helper or presenter | Logic belongs elsewhere |
| Manual `<form>` tag | `form_with` | CSRF protection, conventions |
| `<% @products.each do \|p\| %>` + `render` inside | `render collection:` | Performance + caching |
| `<div onclick="...">` | Stimulus controller | Unobtrusive JS |

## Accessibility Checklist

- [ ] All images have `alt` attributes
- [ ] Form fields have associated `<label>` elements
- [ ] Interactive elements are keyboard-accessible
- [ ] Color contrast meets WCAG AA (4.5:1 for text)
- [ ] ARIA landmarks on major page regions
- [ ] Skip-to-content link present

## Output Format

When reporting on view quality, use:

```
## View Analysis: [file_path]

**Issues Found:**
- [severity] description — suggested fix

**Recommendations:**
1. actionable recommendation
2. ...
```
