---
name: rails-graphql-patterns
description: Analyzes and recommends GraphQL patterns for Rails using graphql-ruby including schema design, types, resolvers, mutations, subscriptions, DataLoader, and query complexity. Use when building GraphQL APIs, defining types, writing mutations, optimizing N+1 queries, or structuring app/graphql. NOT for REST API controllers, ActiveRecord queries outside GraphQL, or Turbo Stream responses.
allowed-tools: Read, Grep, Glob
---

# Rails GraphQL Patterns

Analyze and recommend patterns for building GraphQL APIs in Rails with graphql-ruby.

## Quick Reference

| Component | Purpose | Location |
|-----------|---------|----------|
| Types | Define object shapes | `app/graphql/types/` |
| QueryType | Read entry points | `app/graphql/types/query_type.rb` |
| Mutations | Write operations | `app/graphql/mutations/` |
| Resolvers | Reusable query logic | `app/graphql/resolvers/` |
| Sources | DataLoader batch loading | `app/graphql/sources/` |
| Subscriptions | Real-time updates | `app/graphql/types/subscription_type.rb` |

## Supporting Documentation

- [patterns.md](patterns.md) - Complete GraphQL patterns with detailed examples

## Core Principles

1. **Type safety**: Define explicit types; be intentional with `null: true/false`
2. **Batch loading**: Always use DataLoader to prevent N+1 queries
3. **Field-level authorization**: Check permissions per field, not just per query
4. **Structured errors**: Return error arrays in mutation payloads, not exceptions
5. **Complexity limits**: Set `max_complexity` and `max_depth` on the schema

## Type Definition

```ruby
module Types
  class UserType < Types::BaseObject
    field :id, ID, null: false
    field :email, String, null: false
    field :name, String
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :posts, [Types::PostType]
    field :posts_count, Integer, null: false

    def posts_count
      object.posts.size  # uses counter_cache
    end
  end
end
```

## Mutation Pattern

```ruby
module Mutations
  class CreatePost < BaseMutation
    argument :title, String, required: true
    argument :content, String, required: true

    field :post, Types::PostType
    field :errors, [String], null: false

    def resolve(title:, content:)
      authenticate!
      post = context[:current_user].posts.build(title: title, content: content)
      if post.save
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    end
  end
end
```

## DataLoader for N+1 Prevention

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

## Connection Types (Pagination)

```ruby
module Types
  class PostConnectionType < Types::BaseConnection
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

## Subscriptions

```ruby
module Types
  class SubscriptionType < Types::BaseObject
    field :post_created, Types::PostType, null: false do
      argument :user_id, ID, required: false
    end

    def post_created(user_id: nil)
      return object unless user_id
      object if object.user_id.to_s == user_id.to_s
    end
  end
end

# Trigger from model or service
MyAppSchema.subscriptions.trigger("postCreated", {}, post)
```

## Query Complexity Analysis

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

## Caching

Use `Rails.cache.fetch` in resolver methods for expensive computations. Cache keys should include object ID and association name. See [patterns.md](patterns.md) for examples.

## Testing

Execute queries against the schema directly: `MyAppSchema.execute(query, variables:, context:)`. Assert on `result["data"]` and `result["errors"]`. See [patterns.md](patterns.md) for full test patterns.

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| N+1 in resolvers | Slow queries, DB overload | Use DataLoader / Sources |
| No complexity limits | Abusive queries crash server | Set `max_complexity` and `max_depth` |
| Raising exceptions in mutations | Breaks GraphQL error contract | Return `{ errors: [...] }` in payload |
| Authorization only at query root | Fields leak data | Check auth at field level |
| Exposing ActiveRecord directly | Schema coupled to DB | Define explicit GraphQL types |
| Fat resolvers with business logic | Hard to test | Delegate to service objects |
| No pagination on list fields | Unbounded result sets | Use connection types |

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
- Log resolver errors with query context for debugging
- Never expose internal error details to clients in production
