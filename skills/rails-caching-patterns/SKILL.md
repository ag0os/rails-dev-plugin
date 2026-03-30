---
name: rails-caching-patterns
description: Caching patterns for Rails applications including fragment caching, low-level caching, HTTP caching, Russian doll caching, and cache invalidation strategies. Automatically invoked when working with Rails.cache, cache stores, stale?/fresh_when, fragment caching, cache keys, or performance optimization through caching. Triggers on "cache", "caching", "Rails.cache", "fragment cache", "Russian doll", "stale?", "fresh_when", "cache key", "cache store", "Redis cache", "Solid Cache", "memcached", "ETag", "cache invalidation", "cache bust". NOT for CDN configuration (use rails-devops-patterns) or database query optimization (use rails-model-patterns).
allowed-tools: Read, Grep, Glob
---

# Rails Caching Patterns

Analyze and recommend caching strategies across all layers of a Rails application.

Follow standard Rails conventions for basic `cache` helper, `Rails.cache.fetch`, `stale?`/`fresh_when`, and `expires_in`. Focus on the opinionated patterns below.

See [patterns.md](patterns.md) for detailed code examples.

## Quick Reference

| Pattern | Use When |
|---------|----------|
| Russian doll + `touch: true` | Nested associations — inner updates bust outer caches |
| `race_condition_ttl` | High-traffic keys — prevents cache stampede on expiry |
| `fetch_multi` | Multiple cache reads — one round-trip instead of N |
| Collection caching (`cached: true`) | Rendering collections — uses `read_multi` internally |
| Solid Cache vs Redis | **Omakase:** Solid Cache. **Service-oriented:** Redis |
| Composite cache keys | Varying by user, locale, or version |

## Profile-Aware Cache Store

| Profile | Store | Why |
|---------|-------|-----|
| **Omakase** | `config.cache_store = :solid_cache_store` | No external deps, database-backed, Rails 8 default |
| **Service-oriented** | `config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }` | Already running Redis for Sidekiq/ActionCable |
| **Development** | `:memory_store` | Toggle with `bin/rails dev:cache` |

## Core Principles

1. **Cache the expensive thing closest to where it's used**: HTTP > fragment > low-level
2. **Key-based expiration over manual invalidation**: Let `cache_key_with_version` change when data changes
3. **Measure before caching**: Use `rack-mini-profiler` or server timing — don't cache what isn't slow
4. **Russian doll from outside in**: `touch: true` propagates changes up the association chain
5. **Never cache user-specific content in shared caches** without varying the key

## Russian Doll Caching

The critical setup: `touch: true` on associations propagates `updated_at` changes upward.

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end

class Post < ApplicationRecord
  belongs_to :category, touch: true
  has_many :comments, dependent: :destroy
end
```

```erb
<% cache @category do %>
  <h2><%= @category.name %></h2>
  <% @category.posts.includes(:comments).each do |post| %>
    <% cache post do %>
      <h3><%= post.title %></h3>
      <% post.comments.each do |comment| %>
        <% cache comment do %>
          <p><%= comment.body %></p>
        <% end %>
      <% end %>
    <% end %>
  <% end %>
<% end %>
```

**How it works**: Comment updated -> `post.touch` -> `category.touch`. Only changed fragments re-render; unchanged siblings hit cache.

## Race Condition TTL

Prevents cache stampede when many requests hit an expired key simultaneously. First request recalculates; others get stale data for the TTL window:

```ruby
Rails.cache.fetch("dashboard/stats", expires_in: 5.minutes, race_condition_ttl: 30.seconds) do
  DashboardStats.calculate  # Expensive
end
```

## Collection Caching with `read_multi`

One cache round-trip for the entire collection:

```erb
<%= render partial: "products/product", collection: @products, cached: true %>
```

## `fetch_multi` for Batch Cache Reads

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

## Composite Cache Keys

```ruby
# Varies by multiple factors
cache [@product, current_user.locale, "v2"] do ... end

# Conditional caching — skip cache for admins
cache_if !current_user&.admin?, @product do ... end
```

## Write-Through Cache

Update cache proactively when data changes instead of waiting for expiry:

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

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Caching without measuring | Profile first with `rack-mini-profiler` |
| Manual invalidation everywhere | Use key-based expiration (`cache_key_with_version`) |
| No `touch: true` with Russian doll | Parent cache never busts — add `touch: true` |
| Long `expires_in` without `race_condition_ttl` | Cache stampede on expiry — add `race_condition_ttl: 30.seconds` |
| `Rails.cache.fetch` in a loop | N round-trips — use `fetch_multi` |
| Caching nil results | Thundering herd — return null object or empty result |
| User-specific content in shared cache | Data leaks — include user/session in cache key |

## Output Format

When recommending caching, provide:
1. **Layer** — which caching layer applies (HTTP, fragment, low-level)
2. **Implementation** — code with cache keys and expiration
3. **Invalidation strategy** — how/when the cache busts
4. **Measurement** — how to verify the cache is helping
