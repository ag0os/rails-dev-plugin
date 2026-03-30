---
name: rails-views-patterns
description: Analyzes Rails view templates, partials, layouts, helpers, and form patterns for best practices. Use when reviewing ERB templates, improving view performance with fragment caching, fixing form helpers, organizing partials, adding accessibility attributes, or evaluating collection rendering. NOT for Stimulus/Turbo logic (use hotwire-patterns), controller concerns, or API-only responses.
allowed-tools: Read, Grep, Glob
---

# Rails Views Patterns

Follow standard Rails view conventions for ERB templates, partials, layouts, helpers, `form_with`, and `content_for` regions. This skill covers only the opinionated and non-obvious additions.

See [patterns.md](patterns.md) for detailed examples.

## Quick Reference

| Area | Key Rule | Anti-Pattern |
|------|----------|--------------|
| Partials | Extract reusable components | Duplicated HTML across views |
| Collections | Always use `render collection:` | Manual loops with `each` |
| Caching | Fragment cache expensive renders | No caching on collection renders |
| Accessibility | Semantic HTML + ARIA | `<div>` soup without roles |
| Logic | Presenter or helper for complex logic | Conditionals in templates |

## Core Principles

1. **Views are for presentation only** -- no queries, no mutations, no business logic
2. **Semantic HTML5** -- use `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>` instead of generic `<div>` wrappers
3. **Always `render collection:`** -- never `each` + `render` inside the loop (performance + enables collection caching)
4. **Cache aggressively** -- fragment caching with `cached: true` on collections, Russian-doll nesting for parent/child

## Presenter Objects (Profile-Aware)

**Omakase:** Extract complex view logic into plain Ruby presenter objects -- not ViewComponents.

```ruby
# app/presenters/dashboard_presenter.rb (or app/models/ for omakase)
class DashboardPresenter
  attr_reader :user, :period

  def initialize(user:, period: Date.current.all_month)
    @user = user
    @period = period
  end

  def grouped_activities
    user.activities.where(date: period).group_by(&:category).sort_by { |cat, _| cat.position }
  end

  def empty? = user.activities.where(date: period).none?
  def title  = "#{user.name}'s Dashboard"
end
```

```erb
<%# Clean view -- no logic %>
<h1><%= @presenter.title %></h1>
<% @presenter.grouped_activities.each do |category, activities| %>
  <%= render partial: "activity", collection: activities, cached: true %>
<% end %>
```

**Service-oriented:** ViewComponents are preferred when the `view_component` gem is present. See [patterns.md](patterns.md) for the ViewComponent pattern.

## Anti-Patterns

| Bad | Good | Why |
|-----|------|-----|
| `<% @users = User.all %>` in view | Pass `@users` from controller | Views must not query |
| `<% if user.admin? && user.active? && ... %>` | Extract to helper or presenter | Logic belongs elsewhere |
| `<% @products.each do \|p\| %>` + `render` | `render partial:, collection:` | Performance + caching |
| `<div onclick="...">` | Stimulus controller | Unobtrusive JS |

## Accessibility Checklist

- [ ] All images have meaningful `alt` attributes (empty `alt=""` for decorative images)
- [ ] Form fields have associated `<label>` elements (not just placeholder text)
- [ ] Interactive elements are keyboard-accessible
- [ ] Color contrast meets WCAG AA (4.5:1 for text, 3:1 for large text)
- [ ] ARIA landmarks on major page regions (`role="navigation"`, `role="main"`, etc.)
- [ ] Skip-to-content link as first focusable element in `<body>`
- [ ] Error messages linked via `aria-describedby` and announced with `role="alert"`
- [ ] Flash messages wrapped in `aria-live="polite"` region

## Output Format

When reporting on view quality, use:

```
## View Analysis: [file_path]

**Issues Found:**
- [severity] description -- suggested fix

**Recommendations:**
1. actionable recommendation
2. ...
```
