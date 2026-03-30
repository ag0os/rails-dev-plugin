---
name: rails-caching-patterns
description: Caching patterns for Rails applications including fragment caching, low-level caching, HTTP caching, Russian doll caching, and cache invalidation strategies. Automatically invoked when working with Rails.cache, cache stores, stale?/fresh_when, fragment caching, cache keys, or performance optimization through caching. Triggers on "cache", "caching", "Rails.cache", "fragment cache", "Russian doll", "stale?", "fresh_when", "cache key", "cache store", "Redis cache", "Solid Cache", "memcached", "ETag", "cache invalidation", "cache bust". NOT for CDN configuration (use rails-devops-patterns) or database query optimization (use rails-model-patterns).
allowed-tools: Read, Grep, Glob
---

# Rails Caching Patterns

Analyze and recommend caching strategies across all layers of a Rails application — from HTTP to fragment to low-level caching.

See [patterns.md](patterns.md) for detailed code examples.

## Quick Reference

| Layer | Technique | Where | Cache Hit Avoids |
|-------|-----------|-------|-----------------|
| HTTP | `stale?` / `fresh_when` | Controller | Entire response render |
| Fragment | `cache do...end` | View | Partial/template render |
| Low-level | `Rails.cache.fetch` | Anywhere | Expensive computation |
| Query | Counter cache, memoization | Model | Database query |
| Collection | `render collection:, cached: true` | View | Per-item partial render |

## Profile-Aware Cache Store

| Profile | Default Store | Config |
|---------|--------------|--------|
| **Omakase** | Solid Cache | `config.cache_store = :solid_cache_store` |
| **Service-oriented** | Redis | `config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }` |
| **Development** | Memory store | `config.cache_store = :memory_store` |

## Core Principles

1. **Cache the expensive thing closest to where it's used**: HTTP > fragment > low-level
2. **Key-based expiration over manual invalidation**: Let cache keys change when data changes
3. **Measure before caching**: Don't cache what isn't slow. Use `rack-mini-profiler` or server timing
4. **Russian doll from the outside in**: Outer cache wraps inner caches; `touch: true` propagates changes
5. **Never cache user-specific content in shared caches** without varying the key

## HTTP Caching (Controller)

Prevents the server from rendering at all when content hasn't changed:

```ruby
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])

    # Returns 304 Not Modified if product hasn't changed
    if stale?(@product)
      respond_to do |format|
        format.html
        format.json { render json: @product }
      end
    end
  end

  def index
    @products = Product.active.order(:updated_at)

    # fresh_when — simpler API when you just render
    fresh_when @products
  end
end
```

## Fragment Caching (Views)

```erb
<%# Basic %>
<% cache @product do %>
  <%= render "products/card", product: @product %>
<% end %>

<%# Collection — uses read_multi for one cache round-trip %>
<%= render partial: "products/product", collection: @products, cached: true %>

<%# Conditional — only cache for anonymous users %>
<% cache_if current_user.nil?, @product do %>
  <%= render "products/card", product: @product %>
<% end %>
```

### Russian Doll Caching

Nested caches — inner updates only bust the inner fragment. Requires `touch: true` on child associations.

```ruby
class Product < ApplicationRecord
  belongs_to :category, touch: true
end
```

See [patterns.md](patterns.md) for Russian doll view examples, collection ETags, and conditional caching.

## Low-Level Caching

For expensive computations, external API calls, or complex queries:

```ruby
class Product < ApplicationRecord
  def expensive_stats
    Rails.cache.fetch("product/#{id}/stats", expires_in: 1.hour) do
      {
        avg_rating: reviews.average(:rating)&.round(2),
        review_count: reviews.count,
        sales_rank: calculate_sales_rank
      }
    end
  end
end
```

### Race Condition TTL

Prevents cache stampede when many requests hit an expired key simultaneously:

```ruby
Rails.cache.fetch("dashboard/stats", expires_in: 5.minutes, race_condition_ttl: 30.seconds) do
  DashboardStats.calculate
end
```

## Cache Key Strategies

```ruby
# Automatic (preferred) — changes when record updates
@product.cache_key_with_version
# => "products/123-20240115123456789000"

# Composite — varies by multiple factors
cache [@product, current_user.locale, "v2"] do ...end

# Template digest — Rails auto-includes in cache key
cache @product do ...end  # Changing the template busts the cache
```

## Memoization (Request-Level Cache)

```ruby
class User < ApplicationRecord
  def permissions
    @permissions ||= roles.flat_map(&:permissions).uniq
  end
end
```

See [patterns.md](patterns.md) for cache invalidation strategies, fetch_multi, write-through cache, counter caches, and performance measurement.

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Caching without measuring | Premature optimization | Profile first with rack-mini-profiler |
| Manual cache invalidation everywhere | Stale data, complex code | Use key-based expiration (cache_key_with_version) |
| Caching user-specific content globally | Data leaks between users | Include user/session in cache key, or use `cache_if` |
| No `touch: true` with Russian doll | Parent cache never busts | Add `touch: true` on child associations |
| Long `expires_in` without race_condition_ttl | Cache stampede on expiry | Add `race_condition_ttl: 30.seconds` |
| Caching nil results | Thundering herd on missing data | Return a null object or empty result, not nil |
| `Rails.cache.fetch` in a loop | N cache round-trips | Use `Rails.cache.fetch_multi` |

## Output Format

When recommending caching, provide:
1. **Layer** — which caching layer applies (HTTP, fragment, low-level)
2. **Implementation** — code with cache keys and expiration
3. **Invalidation strategy** — how/when the cache busts
4. **Measurement** — how to verify the cache is helping
