# Caching Patterns — Detailed Reference

Follow standard Rails conventions for basic `cache` helper syntax, `Rails.cache.fetch`, `stale?`/`fresh_when`, `expires_in`, ETags, and `Cache-Control` headers. This file covers opinionated and non-obvious patterns.

## Cache Store Configuration

### Solid Cache (Omakase — Rails 8 default)

Database-backed cache. No external dependencies.

```ruby
# config/environments/production.rb
config.cache_store = :solid_cache_store

# config/solid_cache.yml
default: &default
  database: cache
  store_options:
    max_age: 1.week
    max_size: 256.megabytes
    namespace: null

production:
  <<: *default

# config/database.yml — add cache database
production:
  primary:
    <<: *default
  cache:
    <<: *default
    database: app_cache
    migrations_paths: db/cache_migrate
```

### Redis Cache Store (Service-Oriented)

```ruby
config.cache_store = :redis_cache_store, {
  url: ENV.fetch("REDIS_URL", "redis://localhost:6379/1"),
  expires_in: 1.hour,
  namespace: "app_cache",
  error_handler: -> (method:, returning:, exception:) {
    Rails.logger.error("Redis Cache Error: #{exception.message}")
    Sentry.capture_exception(exception) if defined?(Sentry)
  }
}
```

**Key difference**: Redis error handler prevents cache failures from crashing the app. Always include one.

## Russian Doll Caching (Deep Dive)

### Setup — `touch: true` chain

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end

class Post < ApplicationRecord
  belongs_to :category, touch: true
  has_many :comments, dependent: :destroy
end
```

### View — nested cache blocks

```erb
<% cache @category do %>
  <h2><%= @category.name %></h2>
  <% @category.posts.includes(:comments).each do |post| %>
    <% cache post do %>
      <h3><%= post.title %></h3>
      <% post.comments.each do |comment| %>
        <% cache comment do %>
          <p><%= comment.body %> — <%= comment.author.name %></p>
        <% end %>
      <% end %>
    <% end %>
  <% end %>
<% end %>
```

**How it works**:
1. Comment updated -> `post.touch` -> `category.touch`
2. Comment cache busts (its `cache_key_with_version` changed)
3. Post cache busts (its `updated_at` was touched)
4. Category cache busts (its `updated_at` was touched)
5. On next request, only changed parts re-render; unchanged siblings hit cache

## Race Condition TTL

Prevents thundering herd when many requests hit an expired key:

```ruby
# Without race_condition_ttl: 100 requests all recalculate simultaneously
# With: first request recalculates, others get stale data for 30s
Rails.cache.fetch("expensive_report", expires_in: 5.minutes, race_condition_ttl: 30.seconds) do
  ExpensiveReport.generate
end
```

## Collection Caching with `read_multi`

Single cache round-trip for the entire collection:

```erb
<%= render partial: "products/product", collection: @products, cached: true %>
```

Rails calls `read_multi` on the cache store, fetching all cached partials in one operation.

## `fetch_multi` for Batch Cache Reads

```ruby
class Product < ApplicationRecord
  def self.with_cached_stats(products)
    keys = products.map { |p| "product/#{p.id}/stats" }

    cached = Rails.cache.fetch_multi(*keys, expires_in: 1.hour) do |key|
      product_id = key.split("/")[1].to_i
      calculate_stats_for(product_id)
    end

    products.each do |product|
      product.cached_stats = cached["product/#{product.id}/stats"]
    end
  end
end
```

## Write-Through Cache

Update cache proactively when data changes:

```ruby
class Product < ApplicationRecord
  after_commit :update_cache, on: [:create, :update]
  after_commit :clear_cache, on: :destroy

  def cached_stats
    Rails.cache.fetch(stats_cache_key, expires_in: 1.day) do
      calculate_stats
    end
  end

  private

  def update_cache
    Rails.cache.write(stats_cache_key, calculate_stats, expires_in: 1.day)
  end

  def clear_cache
    Rails.cache.delete(stats_cache_key)
  end

  def stats_cache_key = "product/#{id}/stats"
end
```

## Custom Cache Key Strategies

```ruby
# Composite keys — vary by multiple factors
cache [@product, current_user.locale, "v2"] do ... end

# Conditional caching — skip for admins
cache_if !current_user&.admin?, @product do ... end

# With Turbo Frames
turbo_frame_tag @product do
  cache @product do
    render "products/detail", product: @product
  end
end
```

## Pattern-Based Deletion (Redis only)

```ruby
Rails.cache.delete_matched("product/#{id}/*")
# Only works with Redis cache store — not available with Solid Cache
```

## Performance Measurement

### Cache Hit Rate Monitoring

```ruby
ActiveSupport::Notifications.subscribe("cache_read.active_support") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  hit = event.payload[:hit]
  key = event.payload[:key]

  StatsD.increment("cache.#{hit ? 'hit' : 'miss'}")
  Rails.logger.debug("[Cache] #{hit ? 'HIT' : 'MISS'}: #{key}") if Rails.env.development?
end
```

### Server Timing Headers

```ruby
# config/application.rb
config.server_timing = true
# Adds Server-Timing header visible in browser DevTools > Network > Timing
```
