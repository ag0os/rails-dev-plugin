# Safe Migration Patterns

Generate standard Rails migrations (create_table, add_column, add_index, references, foreign keys) following Rails conventions. This document covers production safety patterns and non-obvious decisions.

## Production Safety Patterns

### Adding Columns to Large Tables

```ruby
# Safe: add nullable column first, backfill, then constrain
# Step 1: Add column allowing null
add_column :users, :role, :string

# Step 2: Backfill in batches (separate migration or rake task)
User.in_batches.update_all(role: 'member')

# Step 3: Add constraints
change_column_null :users, :role, false
change_column_default :users, :role, 'member'
```

For small tables, `add_column :users, :admin, :boolean, default: false, null: false` is safe in one step.

### Adding Indexes Without Downtime (PostgreSQL)

```ruby
class AddIndexToPostsTitle < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!

  def change
    add_index :posts, :title, algorithm: :concurrently
  end
end
```

Always use `disable_ddl_transaction!` with `algorithm: :concurrently`.

### Safe Column Removal (Two-Deploy Process)

```ruby
# Deploy 1: Tell Rails to ignore the column
class User < ApplicationRecord
  self.ignored_columns += [:old_column]
end

# Deploy 2: Remove column
remove_column :users, :old_column, :string
```

Never remove a column that the running app still references.

### Dangerous Operations Checklist

| Operation | Risk | Safe Alternative |
|-----------|------|-----------------|
| Remove column | App crash if still referenced | `ignored_columns` first |
| Rename column | App crash | Add new, migrate data, remove old |
| Change column type | Data loss | Add new column, copy data |
| Add NOT NULL to existing column | Fails if nulls exist | Backfill first, then constrain |
| Add index on large table | Table lock | `algorithm: :concurrently` (PG) |

## PostgreSQL-Specific Patterns

```ruby
# UUID primary keys
create_table :posts, id: :uuid do |t|
  t.string :title
end

# JSONB with GIN index
add_column :users, :preferences, :jsonb, default: {}
add_index :users, :preferences, using: :gin

# Array columns
add_column :posts, :tags, :string, array: true, default: []
add_index :posts, :tags, using: :gin

# Native enum
create_enum :post_status, ['draft', 'published', 'archived']
add_column :posts, :status, :enum, enum_type: :post_status, default: 'draft'

# Partial index (index only matching rows)
add_index :posts, :published_at, where: "published = true"

# Expression index
add_index :users, 'lower(email)', unique: true
```

## Data Migrations

### Small tables — inline in migration:

```ruby
def up
  add_column :users, :slug, :string
  User.find_each { |u| u.update_column(:slug, u.name.parameterize) }
  add_index :users, :slug, unique: true
  change_column_null :users, :slug, false
end
```

### Large tables — use a background job or rake task:

```ruby
# Migration: add column + index only
# Separate rake task: User.in_batches(of: 1000) { |batch| batch.update_all(...) }
```

## Index Strategy

- Always index foreign keys (Rails does this automatically with `t.references`)
- Index columns used in `WHERE`, `ORDER BY`, and `GROUP BY`
- Use composite indexes for common multi-column queries (leftmost prefix rule applies)
- Prefer partial indexes when queries always filter on a condition

## Key Rules

- Every migration should be reversible (use `change` when possible, `up/down` when not)
- Always add foreign key constraints for referential integrity
- Use `reversible` block for raw SQL that needs both directions
- Test migrations with `rails db:migrate && rails db:rollback` before deploying
