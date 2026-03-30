# Caching Patterns — Detailed Reference

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
# config/environments/production.rb
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

### Memory Store (Development)

```ruby
# config/environments/development.rb
config.cache_store = :memory_store, { size: 64.megabytes }
```

### Toggle Development Caching

```bash
bin/rails dev:cache
# Creates/removes tmp/caching-dev.txt to toggle caching in development
```

## HTTP Caching Patterns

### ETag-Based (Conditional GET)

```ruby
class ArticlesController < ApplicationController
  def show
    @article = Article.find(params[:id])

    # Returns 304 Not Modified if ETag matches
    if stale?(etag: @article, last_modified: @article.updated_at)
      render :show
    end
  end
end
```

### Collection ETag

```ruby
class ArticlesController < ApplicationController
  def index
    @articles = Article.published.recent

    # ETag based on collection content + count
    if stale?(etag: @articles, last_modified: @articles.maximum(:updated_at))
      render :index
    end
  end
end
```

### fresh_when (Simpler API)

When you just render — no conditional logic needed:

```ruby
def show
  @article = Article.find(params[:id])
  fresh_when @article
  # Automatically returns 304 if not modified, otherwise renders :show
end
```

### Public vs Private Cache

```ruby
def show
  @article = Article.find(params[:id])

  if @article.public?
    # Can be cached by CDN/proxies
    expires_in 5.minutes, public: true
    fresh_when @article, public: true
  else
    # Browser-only cache
    fresh_when @article
  end
end
```

### Manual Cache Headers

```ruby
def index
  expires_in 1.hour, public: true
  # Sets: Cache-Control: max-age=3600, public
end

def dashboard
  expires_now
  # Sets: Cache-Control: no-cache
end
```

## Fragment Caching Patterns

### Explicit Cache Key

```erb
<% cache ["v2", @product] do %>
  <%# "v2" prefix lets you bust all caches by changing the version %>
  <%= render "products/card", product: @product %>
<% end %>
```

### Multi-Key Caching

```erb
<% cache [@product, current_user.admin?, I18n.locale] do %>
  <%# Different cache for admins, different cache per locale %>
  <%= render "products/card", product: @product %>
<% end %>
```

### Caching with Turbo Frames

```erb
<%= turbo_frame_tag @product do %>
  <% cache @product do %>
    <%= render "products/detail", product: @product %>
  <% end %>
<% end %>
```

### Conditional Caching

```erb
<%# Don't cache for admins who see extra controls %>
<% cache_if !current_user&.admin?, @product do %>
  <%= render "products/card", product: @product %>
<% end %>

<%# Don't cache if content is frequently updated %>
<% cache_unless @product.draft?, @product do %>
  <%= render "products/card", product: @product %>
<% end %>
```

## Low-Level Caching Patterns

### Basic Fetch

```ruby
class DashboardController < ApplicationController
  def show
    @stats = Rails.cache.fetch("dashboard/stats/#{Current.account.id}", expires_in: 15.minutes) do
      {
        total_orders: Current.account.orders.count,
        revenue_this_month: Current.account.orders.this_month.sum(:total),
        active_users: Current.account.users.active.count
      }
    end
  end
end
```

### Fetch Multi (Batch)

Avoid N cache round-trips:

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

### Write-Through Cache

Update cache when data changes instead of waiting for expiry:

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

  def stats_cache_key
    "product/#{id}/stats"
  end
end
```

### Cache with Race Condition TTL

Prevents thundering herd when many requests hit an expired key:

```ruby
# Without race_condition_ttl: 100 requests all recalculate simultaneously
# With race_condition_ttl: first request recalculates, others get stale data for 30s
Rails.cache.fetch("expensive_report", expires_in: 5.minutes, race_condition_ttl: 30.seconds) do
  ExpensiveReport.generate
end
```

### Cache Versioning

```ruby
# Instead of deleting cache, increment version
class Product < ApplicationRecord
  def cache_version
    "v#{updated_at.to_i}"
  end

  def pricing_data
    Rails.cache.fetch("product/#{id}/pricing/#{cache_version}", expires_in: 1.day) do
      calculate_complex_pricing
    end
  end
end
```

## Russian Doll Caching (Deep Dive)

### Setup

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true  # Updates post.updated_at when comment changes
end

class Post < ApplicationRecord
  belongs_to :category, touch: true  # Updates category.updated_at when post changes
  has_many :comments, dependent: :destroy
end
```

### View

```erb
<%# Category cache wraps post caches %>
<% cache @category do %>
  <h2><%= @category.name %></h2>
  <% @category.posts.includes(:comments).each do |post| %>
    <%# Post cache wraps comment rendering %>
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

**How it works:**
1. Comment updated → `post.touch` → `category.touch`
2. Comment cache busts (its `cache_key_with_version` changed)
3. Post cache busts (its `updated_at` was touched)
4. Category cache busts (its `updated_at` was touched)
5. On next request, only the changed parts re-render; unchanged siblings hit cache

## Cache Invalidation Strategies

### Key-Based Expiration (Preferred)

```ruby
# ActiveRecord objects use cache_key_with_version automatically
cache @product  # Key: products/123-20240115123456
# When product updates, key changes → old cache naturally expires
```

### Explicit Deletion

```ruby
class Product < ApplicationRecord
  after_commit :invalidate_caches, on: [:update, :destroy]

  private

  def invalidate_caches
    Rails.cache.delete("product/#{id}/stats")
    Rails.cache.delete("category/#{category_id}/product_count")
  end
end
```

### Pattern-Based Deletion (Redis only)

```ruby
# Delete all caches matching a pattern
Rails.cache.delete_matched("product/#{id}/*")
# Only works with Redis cache store
```

### Time-Based Expiration

```ruby
Rails.cache.fetch("exchange_rates", expires_in: 6.hours) do
  ExchangeRateService.fetch_current
end
```

## Performance Measurement

### rack-mini-profiler

```ruby
# Gemfile
gem "rack-mini-profiler"

# Shows timing badge on every page with:
# - SQL query count and time
# - View render time
# - Cache hits/misses
```

### Server Timing Headers

```ruby
# config/application.rb
config.server_timing = true
# Adds Server-Timing header visible in browser DevTools > Network > Timing
```

### Cache Hit Rate Monitoring

```ruby
# Custom instrumentation
ActiveSupport::Notifications.subscribe("cache_read.active_support") do |*args|
  event = ActiveSupport::Notifications::Event.new(*args)
  hit = event.payload[:hit]
  key = event.payload[:key]

  StatsD.increment("cache.#{hit ? 'hit' : 'miss'}")
  Rails.logger.debug("[Cache] #{hit ? 'HIT' : 'MISS'}: #{key}") if Rails.env.development?
end
```

## Common Caching Recipes

### Caching Expensive Aggregations

```ruby
class Category < ApplicationRecord
  def product_stats
    Rails.cache.fetch("#{cache_key_with_version}/product_stats", expires_in: 30.minutes) do
      {
        count: products.count,
        avg_price: products.average(:price)&.round(2),
        min_price: products.minimum(:price),
        max_price: products.maximum(:price)
      }
    end
  end
end
```

### Caching External API Responses

```ruby
class WeatherService
  def forecast(city)
    Rails.cache.fetch("weather/#{city}", expires_in: 30.minutes) do
      response = HTTP.get("https://api.weather.com/forecast", params: { city: city })
      JSON.parse(response.body)
    end
  end
end
```

### Caching Rendered Partials in Helpers

```ruby
module DashboardHelper
  def cached_sidebar
    Rails.cache.fetch("sidebar/#{Current.user.cache_key_with_version}", expires_in: 10.minutes) do
      render("shared/sidebar")
    end
  end
end
```
