# ActiveRecord Association Patterns

Generate standard Rails associations (`belongs_to`, `has_many`, `has_one`, `has_many :through`, polymorphic) following Rails conventions. This document covers decision guidance and advanced patterns only.

## Decision Guidance

### has_many :through vs has_and_belongs_to_many

Always prefer `has_many :through`. Use HABTM only for trivial join tables that will never need attributes, callbacks, or validations on the join record. In practice, almost always use `:through`.

### dependent Option Selection

| Option | Use When |
|--------|----------|
| `:destroy` | Children have callbacks or further dependents |
| `:delete_all` | No callbacks needed, performance matters |
| `:nullify` | Children should survive parent deletion |
| `:restrict_with_error` | Prevent deletion if children exist (soft) |
| `:restrict_with_exception` | Prevent deletion if children exist (hard) |

### Eager Loading Strategy

| Method | Use When |
|--------|----------|
| `includes` | Let Rails decide (default choice) |
| `preload` | Separate queries, no filtering on association |
| `eager_load` | Need to filter/sort by association columns |

## Advanced Patterns

### Default Association Values

Automatically derive association values from related records or current context.

**Context Awareness:** Check for `app/models/current.rb` to determine if CurrentAttributes is available.

```ruby
# With CurrentAttributes (full pattern)
class Card < ApplicationRecord
  belongs_to :account, default: -> { board.account }
  belongs_to :creator, class_name: "User", default: -> { Current.user }
  belongs_to :board, touch: true
end

# Without CurrentAttributes (still useful for derived values)
class Card < ApplicationRecord
  belongs_to :account, default: -> { board.account }
  # creator must be passed explicitly without Current.user
end
```

**Controller benefit:** `@card = @board.cards.create!(title: params[:title])` — no need to pass account or creator.

### Association Extensions

Add custom methods to association collections for domain-specific operations.

```ruby
class Board < ApplicationRecord
  has_many :accesses, dependent: :delete_all do
    def revise(granted: [], revoked: [])
      transaction do
        grant_to granted
        revoke_from revoked
      end
    end

    def grant_to(users)
      Access.insert_all Array(users).collect { |user|
        { id: ActiveRecord::Type::Uuid.generate,
          board_id: proxy_association.owner.id,
          user_id: user.id,
          account_id: user.account_id }
      }
    end

    def revoke_from(users)
      destroy_by(user: users) unless proxy_association.owner.all_access?
    end
  end
end

# Usage: board.accesses.revise(granted: [user1, user2], revoked: [user3])
```

Use `proxy_association.owner` to access the parent record inside extensions.

### Self-Referential Convenience Methods

Add identity methods for polymorphic code uniformity:

```ruby
class Card < ApplicationRecord
  def card = self  # Allows polymorphic code to call .card uniformly
end

class Comment < ApplicationRecord
  belongs_to :card
end

# Enables: [card, comment, attachment].map(&:card)
```

## Key Rules

- Always specify `dependent:` on every `has_many` and `has_one`
- Always specify `inverse_of` (Rails auto-detects most cases, but be explicit for non-standard names)
- Always add database indexes for foreign keys
- Use `counter_cache: true` for frequently accessed counts
- Use `touch: true` when parent cache depends on child updates
- Use default association values to eliminate controller assignment boilerplate
