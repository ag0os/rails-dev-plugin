# Turbo Patterns — Gotchas, Advanced Patterns, and Version-Aware Features

Follow standard Turbo Drive, Frame, and Stream conventions. This file covers non-obvious patterns, debugging tips, and version-specific features only.

## Morphing (Turbo 8+)

Check Turbo version before using morphing. It requires `@hotwired/turbo-rails` >= 8.0.

```erb
<%# Enable page-level morphing (Turbo 8.0+) %>
<meta name="turbo-refresh-method" content="morph">
<meta name="turbo-refresh-scroll" content="preserve">
```

```erb
<%# Stream-level morphing %>
<%= turbo_stream.replace dom_id(@card, :card_container),
    partial: "cards/container",
    method: :morph,
    locals: { card: @card.reload } %>
```

Benefits over standard replace:
- Preserves element focus and scroll position
- Maintains form input state during updates
- Keeps CSS transitions smooth
- More efficient DOM diffing

Detection: `grep "@hotwired/turbo-rails" package.json` -- if version >= 8.0, morphing is available.

## Turbo Stream Custom Actions

Extend Turbo with domain-specific actions when built-in actions are insufficient:

```javascript
// app/javascript/application.js
import { Turbo } from "@hotwired/turbo-rails"

Turbo.StreamActions.notification = function() {
  const message = this.getAttribute("message")
  const type = this.getAttribute("type") || "info"
  window.showNotification(message, type)
}
```

```erb
<turbo-stream action="notification" message="Saved!" type="success"></turbo-stream>
```

## Nested Frames Pattern

Nested frames enable per-item inline editing within a list:

```erb
<turbo-frame id="posts">
  <% @posts.each do |post| %>
    <turbo-frame id="<%= dom_id(post) %>">
      <%= render post %>
    </turbo-frame>
  <% end %>
</turbo-frame>
```

Clicking "Edit" on one post replaces only that post's frame, leaving the rest intact. The inner frame ID (`dom_id(post)`) must match exactly between index and edit views.

## Infinite Scroll with Lazy Frames

This pattern chains lazy-loaded frames for pagination without JavaScript:

```erb
<div id="posts">
  <%= render @posts %>
</div>

<%= turbo_frame_tag "pagination",
    src: posts_path(page: @page + 1),
    loading: :lazy do %>
  <div class="loading">Loading more...</div>
<% end %>
```

```erb
<%# app/views/posts/index.turbo_frame.erb — response for next page %>
<%= turbo_stream.append "posts" do %>
  <%= render @posts %>
<% end %>

<% if @has_more %>
  <%= turbo_frame_tag "pagination",
      src: posts_path(page: @page + 1),
      loading: :lazy do %>
    <div class="loading">Loading more...</div>
  <% end %>
<% end %>
```

## Permanent Elements

Elements with `data-turbo-permanent` persist across navigations, maintaining state and subscriptions:

```erb
<%# ActionCable connections survive navigation %>
<div id="notifications" data-turbo-permanent>
  <%= turbo_stream_from current_user, "notifications" %>
  <div id="notification-list"></div>
</div>

<%# Media players keep playing %>
<div id="media-player" data-turbo-permanent>
  <audio controls src="<%= @podcast.audio_url %>"></audio>
</div>
```

Requirement: element must have a unique `id` and appear in the same position in the new page's DOM. If the element is missing from the new page, it gets removed.

## Fragment Caching with Turbo Frames

Cache at the frame boundary for optimal performance:

```erb
<% @comments.each do |comment| %>
  <% cache comment do %>
    <%= turbo_frame_tag comment, :container do %>
      <%= render comment %>
    <% end %>
  <% end %>
<% end %>
```

```ruby
# Invalidate parent cache when child changes
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end
```

Include user in cache key for personalized content: `cache [@post, current_user]`.

## User-Scoped Broadcasting

```ruby
# Broadcast to a specific user only
Turbo::StreamsChannel.broadcast_prepend_to(
  [user, "notifications"],
  target: "notifications",
  partial: "notifications/notification",
  locals: { notification: notification }
)
```

```erb
<%# View subscribes to user-scoped stream %>
<%= turbo_stream_from current_user, "notifications" %>
```

## Version Requirements

| Pattern | Turbo Version | Notes |
|---------|---------------|-------|
| Frames, Streams | Any | Core functionality |
| Page Refresh | >= 7.2 | `turbo_stream.refresh` |
| Morphing (`:morph` method) | >= 8.0 | Check package.json version |
| Custom Stream Actions | Any | `Turbo.StreamActions` |
