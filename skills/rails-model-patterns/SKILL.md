---
name: rails-model-patterns
description: Analyzes and recommends ActiveRecord model patterns including associations, validations, scopes, callbacks, migrations, and query optimization. Use when designing models, reviewing schema, adding associations (has_many, belongs_to), writing validations, creating scopes, or planning migrations. NOT for controller logic, routing, view rendering, or service object design.
allowed-tools: Read, Grep, Glob
---

# Rails Model Patterns

Analyze and recommend ActiveRecord model patterns for well-structured Rails applications.

## Supporting Documentation

- [associations.md](associations.md) - Association decision guidance and advanced patterns
- [validations.md](validations.md) - Validation strategy and custom validators
- [migrations.md](migrations.md) - Safe migration practices for production
- [value-objects.md](value-objects.md) - Value object and Struct/Data patterns

## Core Principles

1. **Organize by convention**: Constants, associations, validations, scopes, callbacks, class methods, instance methods
2. **Database-backed integrity**: Always pair ActiveRecord validations with database constraints (unique index, NOT NULL, check constraints)
3. **Associations always have `:dependent`**: Prevent orphaned records
4. **Use `:inverse_of`**: Ensure bidirectional association consistency
5. **Callbacks**: **Omakase** — use freely for simple, predictable side effects ("whenever X happens, do Y"). **Service-oriented** — minimize; prefer service objects for side effects

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

### Concern-Based Domain Composition (Omakase)

**Omakase profile:** Concerns are the primary tool for decomposing models — for domain logic, not just shared behavior. Slice by trait/role: `Triageable`, `Postponable`, `Closeable`.

```ruby
# app/models/concerns/closeable.rb
module Closeable
  extend ActiveSupport::Concern
  included do
    has_one :closure, dependent: :destroy
    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def closed? = closure.present?
  def close!(by:, reason: nil) = create_closure!(creator: by, reason: reason)
  def reopen! = closure&.destroy!
end

# app/models/card.rb — composed from concerns
class Card < ApplicationRecord
  include Closeable
  include Triageable
  include Postponable
end
```

### POROs for Domain Operations (Omakase)

**Omakase profile:** Plain Ruby objects for operations that don't fit a model. Not "service objects" — just objects, often in `app/models/`.

```ruby
# app/models/signup.rb
class Signup
  include ActiveModel::Model
  attr_accessor :name, :email, :password
  validates :name, :email, :password, presence: true

  def save
    return false unless valid?
    User.create!(name: name, email: email, password: password)
  end
end
```

### Counter Cache with Default Lambda

```ruby
# migration: add_column :users, :posts_count, :integer, default: 0, null: false
belongs_to :user, counter_cache: true
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Fat models (500+ lines) | Hard to maintain | **Omakase:** extract concerns. **Service-oriented:** extract service objects |
| Callbacks with complex orchestration | Hard to reason about | **Omakase:** keep callbacks simple; use model methods for multi-step workflows. **Service-oriented:** use service objects |
| Missing `:dependent` on associations | Orphaned records | Always specify `:dependent` |
| `default_scope` | Confusing query behavior | Use named scopes instead |
| Validations without DB constraints | Data integrity gaps | Add both layers |
| Boolean flags for state | No audit trail, no metadata | Use state-as-relationship pattern (Closure, Pin, Triage models) |

## Output Format

When analyzing or creating models, provide:
1. **Model file** with proper structure and organization
2. **Migration file** with indexes, constraints, and safe rollback
3. **Test outline**: **Omakase** — Minitest with fixtures. **Service-oriented** — RSpec with factories
4. **Performance notes** on indexes and query patterns

## Error Handling

- Use `save` (returns false) for user-facing forms; `save!` (raises) inside transactions
- Wrap multi-model operations in `ActiveRecord::Base.transaction`
- Handle `ActiveRecord::RecordInvalid`, `ActiveRecord::RecordNotFound`, `ActiveRecord::StaleObjectError`
- Use optimistic locking (`lock_version`) for concurrent updates
