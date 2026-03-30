---
name: rails-graphql-patterns
description: Analyzes and recommends GraphQL patterns for Rails using graphql-ruby including schema design, types, resolvers, mutations, subscriptions, DataLoader, and query complexity. Use when building GraphQL APIs, defining types, writing mutations, optimizing N+1 queries, or structuring app/graphql. NOT for REST API controllers, ActiveRecord queries outside GraphQL, or Turbo Stream responses.
allowed-tools: Read, Grep, Glob
---

# Rails GraphQL Patterns

Analyze and recommend patterns for building GraphQL APIs in Rails with graphql-ruby.

Follow standard graphql-ruby conventions for type definitions, input types, enum types, query types, and basic mutations. This skill focuses on non-obvious patterns and opinionated decisions.

## Core Principles

1. **Batch loading**: Always use DataLoader to prevent N+1 queries — never resolve associations directly
2. **Structured errors**: Return error arrays in mutation payloads, not exceptions
3. **Field-level authorization**: Check permissions per field, not just per query
4. **Complexity limits**: Set `max_complexity` and `max_depth` on the schema
5. **Null annotations**: Be intentional with `null: true/false` on every field

## DataLoader for N+1 Prevention

Every association field MUST use DataLoader. Never call `object.association` directly in a resolver.

```ruby
class Sources::RecordLoader < GraphQL::Dataloader::Source
  def initialize(model_class, column: :id)
    @model_class = model_class
    @column = column
  end

  def fetch(ids)
    records = @model_class.where(@column => ids).index_by { |r| r.send(@column) }
    ids.map { |id| records[id] }
  end
end

# Usage in type
def author
  dataloader.with(Sources::RecordLoader, User).load(object.user_id)
end
```

See [patterns.md](patterns.md) for AssociationLoader and CountLoader variants.

## Connection Types (Pagination)

Always use connection types for list fields — never return unbounded arrays.

```ruby
module Types
  class PostConnectionType < GraphQL::Types::Connection
    edge_type(Types::PostEdgeType)
    field :total_count, Integer, null: false

    def total_count
      object.items.size
    end
  end
end

# In query type
field :posts, Types::PostConnectionType, null: false, connection: true do
  argument :filter, Types::PostFilterInput, required: false
end
```

## Subscriptions — Triggering from Models

```ruby
# Trigger from model after_create_commit, not from mutations
class Post < ApplicationRecord
  after_create_commit :notify_subscribers

  private

  def notify_subscribers
    MyAppSchema.subscriptions.trigger(:post_created, {}, self)
  end
end
```

## Schema Configuration

```ruby
class MyAppSchema < GraphQL::Schema
  max_complexity 300
  max_depth 15
  default_max_page_size 25

  use GraphQL::Dataloader

  rescue_from ActiveRecord::RecordNotFound do |_err, _obj, _args, _ctx, field|
    raise GraphQL::ExecutionError, "#{field.type.unwrap.graphql_name} not found"
  end
end
```

Per-field complexity for expensive operations:

```ruby
field :expensive_field, String do
  complexity 50
end
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| N+1 in resolvers | Use DataLoader / Sources |
| No complexity limits | Set `max_complexity` and `max_depth` |
| Raising exceptions in mutations | Return `{ errors: [...] }` in payload |
| Authorization only at query root | Check auth at field level |
| Exposing ActiveRecord directly | Define explicit GraphQL types |
| Fat resolvers with business logic | Delegate to service objects |
| No pagination on list fields | Use connection types |

## Output Format

When analyzing or creating GraphQL components, provide:
1. **Type/mutation file** with proper null annotations
2. **DataLoader source** if associations are resolved
3. **Schema configuration** (complexity, depth, pagination)
4. **Test outline** executing queries against the schema
5. **Authorization** strategy (context, field-level checks)

## Error Handling

- Mutations return `{ resource: nil, errors: [...] }` on validation failure
- Use `rescue_from` in schema for `RecordNotFound` mapped to `GraphQL::ExecutionError`
- Field-level: return `nil` or raise `GraphQL::ExecutionError` for unauthorized access
- Never expose internal error details to clients in production
