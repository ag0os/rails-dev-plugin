---
name: rails-model-patterns
description: Analyzes and recommends ActiveRecord model patterns including associations, validations, scopes, callbacks, migrations, and query optimization. Use when designing models, reviewing schema, adding associations (has_many, belongs_to), writing validations, creating scopes, or planning migrations. NOT for controller logic, routing, view rendering, or service object design.
allowed-tools: Read, Grep, Glob
---

# Rails Model Patterns

Analyze and recommend ActiveRecord model patterns for well-structured Rails applications.

## Resolve the axis first

Where a model's domain logic and side effects belong forks on **Axis A — logic placement**:

1. If the axes are resolved this session, use them.
2. Otherwise resolve via `rails-stack-profiles` (Axis A — `native` or `extracted`).
3. Then read the matching decomposition guide:
   - `native` → [decomposition.native.md](decomposition.native.md)
   - `extracted` → [decomposition.extracted.md](decomposition.extracted.md)

Default to `native` if the axis cannot be resolved. Everything else in this file is **invariant**.

## Supporting Documentation

- [associations.md](associations.md) - Association decision guidance and advanced patterns
- [validations.md](validations.md) - Validation strategy and custom validators
- [migrations.md](migrations.md) - Safe migration practices for production
- [value-objects.md](value-objects.md) - Value object and Struct/Data patterns
- [decomposition.native.md](decomposition.native.md) / [decomposition.extracted.md](decomposition.extracted.md) - where domain logic and side effects go, per Axis A

## Core Principles

1. **Organize by convention**: Constants, associations, validations, scopes, callbacks, class methods, instance methods
2. **Database-backed integrity**: Always pair ActiveRecord validations with database constraints (unique index, NOT NULL, check constraints)
3. **Associations always have `:dependent`**: Prevent orphaned records
4. **Use `:inverse_of`**: Ensure bidirectional association consistency
5. **Callbacks**: Posture depends on Axis A — read the decomposition guide above. Universally: a callback is for one predictable effect, never multi-step orchestration.

## Model Structure Template

Follow standard Rails model organization. Key ordering: constants, associations, validations, scopes, callbacks, class methods, instance methods, private methods. Generate standard ActiveRecord associations, validations, and scopes following Rails conventions.

## Key Patterns

### State as Relationships (not booleans/enums)

Instead of a `closed` boolean, use a related model. Stores who, when, and why.

```ruby
class Card < ApplicationRecord
  has_one :closure, dependent: :destroy
  def closed? = closure.present?
  def close!(by:, reason: nil) = create_closure!(creator: by, reason: reason)
  def reopen! = closure&.destroy!
end

class Closure < ApplicationRecord
  belongs_to :card
  belongs_to :creator, class_name: "User"
end
```

### Default Association Values

Reduce controller boilerplate using `default:` with lambdas:

```ruby
class Post < ApplicationRecord
  belongs_to :creator, class_name: "User", default: -> { Current.user }
  belongs_to :account, default: -> { Current.account }
end
# Controller: @post = Post.create!(post_params) — no need to assign creator
```

### Counter Cache with Default Lambda

```ruby
# migration: add_column :users, :posts_count, :integer, default: 0, null: false
belongs_to :user, counter_cache: true
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Fat models (500+ lines) | Hard to maintain | Decompose — see the decomposition guide for your axis |
| Callbacks with complex orchestration | Hard to reason about | Simplify the callback — see the decomposition guide for your axis |
| Missing `:dependent` on associations | Orphaned records | Always specify `:dependent` |
| `default_scope` | Confusing query behavior | Use named scopes instead |
| Validations without DB constraints | Data integrity gaps | Add both layers |
| Boolean flags for state | No audit trail, no metadata | Use state-as-relationship pattern (Closure, Pin, Triage models) |

## Output Format

When analyzing or creating models, provide:
1. **Model file** with proper structure and organization
2. **Migration file** with indexes, constraints, and safe rollback
3. **Test outline** in the project's test framework — read it from the `project-conventions` fingerprint (Minitest/RSpec, fixtures/factories); do not infer it from architecture
4. **Performance notes** on indexes and query patterns

## Error Handling

- Use `save` (returns false) for user-facing forms; `save!` (raises) inside transactions
- Wrap multi-model operations in `ActiveRecord::Base.transaction`
- Handle `ActiveRecord::RecordInvalid`, `ActiveRecord::RecordNotFound`, `ActiveRecord::StaleObjectError`
- Use optimistic locking (`lock_version`) for concurrent updates
