---
name: Rails Performance Analyzer
description: Automatically invoked when analyzing performance issues, slow queries, or optimization needs. Triggers on mentions of "slow", "performance", "N+1", "query optimization", "bottleneck", "latency", "speed up", "optimize", "caching", "memory usage", "database performance", "page load time", "response time". Analyzes Rails application performance issues and provides optimization recommendations.
allowed-tools: Read, Grep, Glob, Bash
---

# Rails Performance Analyzer

Systematic performance analysis and optimization for Rails applications.

## When to Use This Skill

- Analyzing slow queries and N+1 problems
- Identifying performance bottlenecks in controllers or models
- Reviewing caching strategies
- Optimizing database queries
- Analyzing memory usage patterns
- Investigating slow page load times
- Profiling application performance

## Performance Analysis Methodology

### 1. Identify Performance Issues

**Common performance problems**:
- **N+1 Queries**: Missing `includes`, `preload`, or `eager_load`
- **Missing Indexes**: Database queries without proper indexes
- **Unnecessary Data Loading**: Loading entire collections when only count needed
- **Cache Misses**: Not utilizing fragment or low-level caching
- **Inefficient Queries**: Complex joins or subqueries that could be optimized
- **Memory Bloat**: Loading too much data into memory at once
- **Unoptimized Assets**: Large JavaScript/CSS files, uncompressed images

### 2. Analyze Query Patterns

Look for these anti-patterns in models and controllers:

```ruby
# ❌ N+1 Query Problem
@posts.each do |post|
  post.author.name  # Triggers query for each post
end

# ✅ Eager Loading
@posts = Post.includes(:author)
@posts.each do |post|
  post.author.name  # No additional queries
end
```

```ruby
# ❌ Loading unnecessary data
if Post.where(published: true).any?
  # Just checking existence
end

# ✅ Use exists?
if Post.where(published: true).exists?
  # More efficient
end
```

### 3. Check for Missing Indexes

Analyze database schema for missing indexes:

```ruby
# Missing index on foreign key
add_column :posts, :author_id, :integer
# Should add:
add_index :posts, :author_id

# Missing composite index for common queries
# If you often query: Post.where(published: true, category_id: 5)
add_index :posts, [:published, :category_id]
```

### 4. Review Caching Opportunities

**Fragment caching**:
```erb
<% cache post do %>
  <%= render post %>
<% end %>
```

**Low-level caching**:
```ruby
def expensive_calculation
  Rails.cache.fetch("calculation/#{id}", expires_in: 12.hours) do
    # Expensive operation here
  end
end
```

**Counter caches**:
```ruby
class Post < ApplicationRecord
  belongs_to :author, counter_cache: true
end

# Access via author.posts_count instead of author.posts.count
```

### 5. Optimize Database Queries

**Select specific columns**:
```ruby
# ❌ Loads all columns
User.all

# ✅ Select only needed columns
User.select(:id, :name, :email)
```

**Use pluck for arrays**:
```ruby
# ❌ Loads full AR objects
User.all.map(&:email)

# ✅ Pluck extracts values directly
User.pluck(:email)
```

**Batch processing**:
```ruby
# ❌ Loads all records into memory
Post.all.each do |post|
  process(post)
end

# ✅ Process in batches
Post.find_each(batch_size: 100) do |post|
  process(post)
end
```

## Performance Testing Commands

Use Bash tool to run performance analysis:

```bash
# Check for N+1 queries with Bullet gem
bundle exec rake bullet:run

# Analyze slow queries
bundle exec rails db:query_log

# Memory profiling
bundle exec derailed bundle:mem

# Benchmarking
bundle exec rails runner 'Benchmark.measure { YourCode.here }'
```

## Quick Performance Checks

### Controllers
- [ ] Are scopes used properly with `policy_scope`?
- [ ] Are associations eager-loaded?
- [ ] Are pagination limits applied?
- [ ] Is caching utilized for expensive operations?

### Models
- [ ] Do all foreign keys have indexes?
- [ ] Are frequently queried columns indexed?
- [ ] Are scopes optimized?
- [ ] Are counter caches used for association counts?

### Views
- [ ] Are fragment caches used for expensive partials?
- [ ] Are database queries minimized in templates?
- [ ] Are assets properly optimized and compressed?

### Background Jobs
- [ ] Are slow operations moved to background jobs?
- [ ] Are job priorities set appropriately?
- [ ] Is batch processing used for large datasets?

## Optimization Recommendations Format

When providing recommendations:

### 1. Issue Identification
- Location (file and line number)
- Severity (High/Medium/Low)
- Performance impact estimation

### 2. Root Cause Analysis
- Why is this slow?
- What's the underlying problem?

### 3. Solution
- Specific code changes
- Before/after comparison
- Expected performance improvement

### 4. Implementation Priority
- Quick wins (low effort, high impact)
- Medium-term improvements
- Long-term refactoring

## Common Optimization Patterns

### N+1 Query Prevention
```ruby
# For has_many associations
includes(:association_name)

# For selective loading
preload(:association_name)

# For joins
eager_load(:association_name)
```

### Database Indexing Strategy
```ruby
# Single column index
add_index :table, :column

# Composite index (order matters!)
add_index :table, [:column1, :column2]

# Partial index
add_index :table, :column, where: "deleted_at IS NULL"

# Unique index
add_index :table, :column, unique: true
```

### Query Optimization
```ruby
# Use select to limit columns
.select(:id, :name)

# Use pluck for simple arrays
.pluck(:id)

# Use exists? for boolean checks
.exists?(id: 123)

# Use find_each for large datasets
.find_each(batch_size: 1000)
```

## Performance Metrics to Track

- **Database queries**: Count and duration
- **Memory usage**: Heap size and allocations
- **Response time**: P50, P95, P99 percentiles
- **Cache hit rate**: Percentage of cache hits
- **Background job queue**: Queue depth and processing time

## Tools Integration

When available, use these tools via Bash:

- **Bullet**: N+1 query detection
- **Rack Mini Profiler**: Request profiling
- **Skylight/New Relic**: APM tools
- **PgHero**: PostgreSQL performance insights
- **Derailed Benchmarks**: Memory and speed benchmarking

## Output Format

Provide analysis in this structure:

```markdown
## Performance Analysis

### Critical Issues (High Priority)
1. [Issue description] - Location: file.rb:line
   - Impact: [Estimated impact]
   - Solution: [Specific fix]

### Medium Priority Issues
...

### Optimization Opportunities
...

### Quick Wins
...
```

Remember: Profile first, optimize second. Always measure the actual performance impact before and after changes.
