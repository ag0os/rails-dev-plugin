---
name: rails-model-patterns
description: Analyzes and recommends ActiveRecord model patterns including associations, validations, scopes, callbacks, migrations, and query optimization. Use when designing models, reviewing schema, adding associations (has_many, belongs_to), writing validations, creating scopes, or planning migrations. NOT for controller logic, routing, view rendering, or service object design.
allowed-tools: Read, Grep, Glob
---

# Rails Model Patterns

Analyze and recommend ActiveRecord model patterns for well-structured Rails applications.

## Quick Reference

| Pattern | Use When | Example |
|---------|----------|---------|
| `belongs_to` | Child references parent | `belongs_to :organization` |
| `has_many` | Parent has multiple children | `has_many :posts, dependent: :destroy` |
| `has_one` | Parent has single child | `has_one :profile, dependent: :destroy` |
| `has_many :through` | Many-to-many via join model | `has_many :tags, through: :taggings` |
| `has_and_belongs_to_many` | Simple M2M, no join attributes | Rarely preferred over :through |
| Value Object | Immutable domain concept | `Money`, `Address`, `DateRange` |
| Counter Cache | Avoid COUNT queries | `belongs_to :user, counter_cache: true` |

## Supporting Documentation

- [associations.md](associations.md) - Association types, options, and advanced patterns
- [validations.md](validations.md) - Validation patterns and custom validators
- [migrations.md](migrations.md) - Safe migration practices and rollback strategies
- [value-objects.md](value-objects.md) - Value object and Struct patterns

## Core Principles

1. **Organize by convention**: Constants, associations, validations, scopes, callbacks, class methods, instance methods
2. **Database-backed integrity**: Pair ActiveRecord validations with database constraints
3. **Associations always have `:dependent`**: Prevent orphaned records
4. **Use `:inverse_of`**: Ensure bidirectional association consistency
5. **Callbacks sparingly**: Prefer service objects for side effects

## Model Structure Template

```ruby
class User < ApplicationRecord
  # Constants
  ROLES = %w[admin member guest].freeze

  # Associations
  belongs_to :organization
  has_many :posts, dependent: :destroy, inverse_of: :author
  has_many :comments, through: :posts
  has_one :profile, dependent: :destroy

  # Validations
  validates :email, presence: true, uniqueness: { case_sensitive: false }
  validates :name, presence: true, length: { maximum: 100 }
  validates :role, inclusion: { in: ROLES }

  # Scopes
  scope :active, -> { where(active: true) }
  scope :recent, -> { order(created_at: :desc) }

  # Callbacks (minimal)
  before_save :normalize_email

  # Class methods
  def self.find_by_email(email) = find_by(email: email.downcase.strip)

  # Instance methods
  def admin? = role == "admin"

  private

  def normalize_email
    self.email = email.downcase.strip
  end
end
```

## Key Patterns

### Scopes Over Class Methods
```ruby
scope :published, -> { where(published: true) }
scope :by_author, ->(user) { where(author: user) }
scope :recent, ->(n = 10) { order(created_at: :desc).limit(n) }
# Chain: Post.published.by_author(user).recent(5)
```

### Database Constraints + Validations
```ruby
# migration
add_index :users, :email, unique: true
change_column_null :users, :email, false

# model
validates :email, presence: true, uniqueness: { case_sensitive: false }
```

### Counter Caches
```ruby
# migration: add_column :users, :posts_count, :integer, default: 0, null: false
belongs_to :user, counter_cache: true
```

### Query Optimization
```ruby
# Avoid N+1
User.includes(:posts, :profile).where(active: true)
# Preload vs Eager Load
User.preload(:posts)        # separate queries
User.eager_load(:posts)     # single LEFT JOIN
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Fat models (500+ lines) | Hard to maintain | Extract concerns, service objects |
| Callbacks calling external services | Hidden side effects, test fragility | Use service objects |
| Missing `:dependent` on associations | Orphaned records | Always specify `:dependent` |
| `default_scope` | Confusing query behavior | Use named scopes instead |
| Validations without DB constraints | Data integrity gaps | Add both layers |
| `has_and_belongs_to_many` | No join model for future attributes | Prefer `has_many :through` |
| String-based SQL in scopes | SQL injection risk | Use Arel or parameterized queries |

## Output Format

When analyzing or creating models, provide:
1. **Model file** with proper structure and organization
2. **Migration file** with indexes, constraints, and safe rollback
3. **Spec outline** covering validations, associations, and scopes
4. **Performance notes** on indexes and query patterns

## Error Handling

- Use `save` (returns false) for user-facing forms; `save!` (raises) inside transactions
- Wrap multi-model operations in `ActiveRecord::Base.transaction`
- Handle `ActiveRecord::RecordInvalid`, `ActiveRecord::RecordNotFound`, `ActiveRecord::StaleObjectError`
- Use optimistic locking (`lock_version`) for concurrent updates
