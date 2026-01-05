# Turbo Patterns for Rails

Patterns for using Turbo Drive, Frames, and Streams in Rails applications.

## Turbo Drive

Turbo Drive intercepts link clicks and form submissions, making navigation feel instant.

### Configuration

```javascript
// Disable for specific links
<a href="/download" data-turbo="false">Download</a>

// Disable for entire sections
<div data-turbo="false">
  <!-- All links/forms here bypass Turbo -->
</div>

// Force full page reload
<meta name="turbo-visit-control" content="reload">
```

### Progress Bar

```javascript
// app/javascript/application.js
import { Turbo } from "@hotwired/turbo-rails"

// Customize progress bar delay (default: 500ms)
Turbo.setProgressBarDelay(100)
```

```css
/* Custom styling */
.turbo-progress-bar {
  background-color: #3b82f6;
  height: 3px;
}
```

## Turbo Frames

Frames scope navigation to a specific part of the page.

### Basic Frame

```erb
<%# app/views/posts/index.html.erb %>
<turbo-frame id="posts">
  <% @posts.each do |post| %>
    <%= render post %>
  <% end %>

  <%= link_to "Load more", posts_path(page: @page + 1) %>
</turbo-frame>
```

### Lazy Loading Frame

```erb
<%# Load content when frame becomes visible %>
<%= turbo_frame_tag "sidebar",
    src: sidebar_path,
    loading: :lazy do %>
  <div class="loading-spinner">Loading...</div>
<% end %>
```

### Frame Navigation Targets

```erb
<%# Navigate within frame %>
<%= link_to "Edit", edit_post_path(post) %>

<%# Break out of frame (full page navigation) %>
<%= link_to "View", post_path(post), data: { turbo_frame: "_top" } %>

<%# Target different frame %>
<%= link_to "Preview", preview_post_path(post), data: { turbo_frame: "preview" } %>
```

### Nested Frames

```erb
<turbo-frame id="posts">
  <% @posts.each do |post| %>
    <turbo-frame id="<%= dom_id(post) %>">
      <%= render post %>
    </turbo-frame>
  <% end %>
</turbo-frame>
```

### Frame with Forms

```erb
<turbo-frame id="<%= dom_id(@post) %>">
  <%= render @post %>
</turbo-frame>

<%# Clicking edit loads form inside frame %>
<%# app/views/posts/edit.html.erb %>
<turbo-frame id="<%= dom_id(@post) %>">
  <%= render "form", post: @post %>
</turbo-frame>
```

## Turbo Streams

Streams update multiple parts of the page from a single response.

### Stream Actions

| Action | Description |
|--------|-------------|
| `append` | Add to end of target |
| `prepend` | Add to beginning of target |
| `replace` | Replace entire target |
| `update` | Replace target's innerHTML |
| `remove` | Remove target element |
| `before` | Insert before target |
| `after` | Insert after target |
| `morph` | Morph target (requires morphing) |
| `refresh` | Refresh the page |

### Stream Response Templates

```erb
<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.prepend "posts", @post %>

<%= turbo_stream.update "post_count" do %>
  <%= Post.count %> posts
<% end %>

<%= turbo_stream.replace "new_post_form" do %>
  <%= render "form", post: Post.new %>
<% end %>
```

### Controller Response

```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)

    respond_to do |format|
      if @post.save
        format.turbo_stream
        format.html { redirect_to @post }
      else
        format.turbo_stream do
          render turbo_stream: turbo_stream.replace(
            "new_post_form",
            partial: "form",
            locals: { post: @post }
          )
        end
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end
end
```

### Inline Stream Response

```ruby
def update
  @post.update(post_params)

  respond_to do |format|
    format.turbo_stream do
      render turbo_stream: [
        turbo_stream.replace(@post),
        turbo_stream.update("flash", partial: "shared/flash")
      ]
    end
  end
end
```

## Real-Time Updates with ActionCable

### Model Broadcasting

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  broadcasts_to ->(post) { "posts" }

  # Or with more control:
  after_create_commit { broadcast_prepend_to "posts" }
  after_update_commit { broadcast_replace_to "posts" }
  after_destroy_commit { broadcast_remove_to "posts" }
end
```

### View Subscription

```erb
<%# Subscribe to stream %>
<%= turbo_stream_from "posts" %>

<div id="posts">
  <%= render @posts %>
</div>
```

### Custom Broadcasting

```ruby
class Post < ApplicationRecord
  after_update_commit :broadcast_notification

  private

  def broadcast_notification
    broadcast_prepend_to(
      "notifications",
      target: "notifications",
      partial: "notifications/notification",
      locals: { message: "#{title} was updated" }
    )
  end
end
```

### User-Specific Streams

```erb
<%# Only this user receives updates %>
<%= turbo_stream_from current_user, "notifications" %>
```

```ruby
# Broadcast to specific user
Turbo::StreamsChannel.broadcast_prepend_to(
  [user, "notifications"],
  target: "notifications",
  partial: "notifications/notification",
  locals: { notification: notification }
)
```

## Common Patterns

### Flash Messages

```erb
<%# app/views/layouts/application.html.erb %>
<div id="flash">
  <%= render "shared/flash" %>
</div>

<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.update "flash" do %>
  <%= render "shared/flash", notice: "Post created!" %>
<% end %>
```

### Pagination with Frames

```erb
<turbo-frame id="posts_page">
  <%= render @posts %>

  <nav>
    <%= link_to "Previous", posts_path(page: @page - 1) if @page > 1 %>
    <%= link_to "Next", posts_path(page: @page + 1) if @has_more %>
  </nav>
</turbo-frame>
```

### Infinite Scroll

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
<%# app/views/posts/index.turbo_frame.erb %>
<%= render @posts %>

<% if @has_more %>
  <%= turbo_frame_tag "pagination",
      src: posts_path(page: @page + 1),
      loading: :lazy do %>
    <div class="loading">Loading more...</div>
  <% end %>
<% end %>
```

### Search with Debounce

```erb
<%= form_with url: search_path, method: :get,
    data: { controller: "debounce", turbo_frame: "results" } do |f| %>
  <%= f.text_field :q,
      data: { action: "input->debounce#search" },
      autofocus: true %>
<% end %>

<turbo-frame id="results">
  <%= render @results %>
</turbo-frame>
```

### Morphing (Turbo 8+)

```erb
<%# Enable page refresh with morphing %>
<meta name="turbo-refresh-method" content="morph">
<meta name="turbo-refresh-scroll" content="preserve">
```

```ruby
# Trigger morph refresh
turbo_stream.refresh
```

## Turbo Stream Custom Actions

```javascript
// app/javascript/application.js
import { Turbo } from "@hotwired/turbo-rails"

// Register custom action
Turbo.StreamActions.notification = function() {
  const message = this.getAttribute("message")
  const type = this.getAttribute("type") || "info"

  // Your notification system
  window.showNotification(message, type)
}
```

```erb
<%# Use custom action %>
<turbo-stream action="notification" message="Saved!" type="success"></turbo-stream>
```

## Best Practices

### Do

- Use frames for scoped navigation (edit forms, tabs, pagination)
- Use streams for multi-element updates
- Keep frame IDs semantic and unique
- Provide fallbacks for non-Turbo browsers
- Use `data-turbo-permanent` for elements that shouldn't change

### Don't

- Don't nest frames unnecessarily
- Don't use streams when a frame would suffice
- Don't forget `status: :unprocessable_entity` for validation errors
- Don't broadcast to everyone when only some users need updates

## Cache Control

```erb
<%# Disable caching for dynamic content %>
<meta name="turbo-cache-control" content="no-cache">

<%# Preserve elements across navigation %>
<div id="audio-player" data-turbo-permanent>
  <audio src="..."></audio>
</div>
```
