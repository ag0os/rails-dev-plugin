# ActiveRecord Association Patterns

Detailed patterns for model associations in Rails.

## Basic Associations

### belongs_to

```ruby
class Post < ApplicationRecord
  belongs_to :user
  belongs_to :category, optional: true  # Allow nil
  belongs_to :author, class_name: 'User', foreign_key: 'author_id'
end
```

**Key Options:**
- `optional: true` - Allow nil (Rails 5+ requires by default)
- `class_name:` - Specify different class name
- `foreign_key:` - Specify custom foreign key
- `counter_cache: true` - Maintain count in parent

### has_many

```ruby
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :nullify
  has_many :drafts, -> { where(published: false) }, class_name: 'Post'

  # With custom scope
  has_many :recent_posts, -> { order(created_at: :desc).limit(5) },
           class_name: 'Post'
end
```

**dependent options:**
- `:destroy` - Delete children with callbacks
- `:delete_all` - Delete children without callbacks (faster)
- `:nullify` - Set foreign key to NULL
- `:restrict_with_error` - Prevent deletion if children exist
- `:restrict_with_exception` - Raise exception if children exist

### has_one

```ruby
class User < ApplicationRecord
  has_one :profile, dependent: :destroy
  has_one :avatar, dependent: :destroy

  # Through association
  has_one :address, through: :profile
end
```

## Many-to-Many Associations

### has_many :through (Recommended)

Use when the join table has additional attributes or you need a model.

```ruby
# Join model with attributes
class Enrollment < ApplicationRecord
  belongs_to :student
  belongs_to :course

  validates :enrolled_at, presence: true
  validates :student_id, uniqueness: { scope: :course_id }
end

class Student < ApplicationRecord
  has_many :enrollments, dependent: :destroy
  has_many :courses, through: :enrollments
end

class Course < ApplicationRecord
  has_many :enrollments, dependent: :destroy
  has_many :students, through: :enrollments
end

# Usage
student.courses << course
student.enrollments.create!(course: course, enrolled_at: Time.current)
```

### has_and_belongs_to_many

Use for simple join tables without additional attributes.

```ruby
class Post < ApplicationRecord
  has_and_belongs_to_many :tags
end

class Tag < ApplicationRecord
  has_and_belongs_to_many :posts
end
```

Migration for join table:
```ruby
create_table :posts_tags, id: false do |t|
  t.belongs_to :post, null: false, foreign_key: true
  t.belongs_to :tag, null: false, foreign_key: true
end

add_index :posts_tags, [:post_id, :tag_id], unique: true
```

## Polymorphic Associations

When a model can belong to multiple other models:

```ruby
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
end

class Post < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

class Photo < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end
```

Migration:
```ruby
create_table :comments do |t|
  t.text :body
  t.references :commentable, polymorphic: true, null: false
  t.timestamps
end
```

## Self-Referential Associations

### Tree Structure (Parent-Child)

```ruby
class Category < ApplicationRecord
  belongs_to :parent, class_name: 'Category', optional: true
  has_many :children, class_name: 'Category', foreign_key: 'parent_id',
           dependent: :destroy

  scope :roots, -> { where(parent_id: nil) }
end
```

### Followers/Following (Many-to-Many Self)

```ruby
class Follow < ApplicationRecord
  belongs_to :follower, class_name: 'User'
  belongs_to :followed, class_name: 'User'

  validates :follower_id, uniqueness: { scope: :followed_id }
end

class User < ApplicationRecord
  has_many :active_follows, class_name: 'Follow',
           foreign_key: 'follower_id', dependent: :destroy
  has_many :passive_follows, class_name: 'Follow',
           foreign_key: 'followed_id', dependent: :destroy

  has_many :following, through: :active_follows, source: :followed
  has_many :followers, through: :passive_follows, source: :follower

  def follow(user)
    following << user unless following.include?(user)
  end

  def unfollow(user)
    following.delete(user)
  end
end
```

## Association Options

### inverse_of

Improves performance by reusing loaded objects:

```ruby
class User < ApplicationRecord
  has_many :posts, inverse_of: :user
end

class Post < ApplicationRecord
  belongs_to :user, inverse_of: :posts
end

# Without inverse_of: 2 queries
user.posts.first.user

# With inverse_of: 1 query (reuses loaded user)
user.posts.first.user  # Same object as user
```

### counter_cache

Maintain count without queries:

```ruby
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
  # Or: counter_cache: :comments_count
end

# Add column to posts
add_column :posts, :comments_count, :integer, default: 0

# Reset counters (one-time)
Post.find_each { |p| Post.reset_counters(p.id, :comments) }
```

### touch

Update parent's timestamp when child changes:

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true
  # Or: touch: :comments_updated_at
end
```

## Eager Loading (Preventing N+1)

```ruby
# N+1 problem
Post.all.each { |p| puts p.user.name }

# Solution 1: includes (preload or eager_load)
Post.includes(:user).each { |p| puts p.user.name }

# Solution 2: preload (separate queries)
Post.preload(:user, :comments).all

# Solution 3: eager_load (single JOIN query)
Post.eager_load(:user).where(users: { active: true })

# Nested eager loading
Post.includes(comments: :user).all
Post.includes(user: { profile: :avatar }).all
```

### When to Use Which

| Method | Use When |
|--------|----------|
| `includes` | Let Rails decide (default choice) |
| `preload` | Separate queries, no filtering on association |
| `eager_load` | Need to filter by association columns |

## Association Scopes

```ruby
class User < ApplicationRecord
  has_many :posts
  has_many :published_posts, -> { where(published: true) }, class_name: 'Post'
  has_many :recent_posts, -> { order(created_at: :desc).limit(10) },
           class_name: 'Post'

  # Scope with parameter (use lambda)
  has_many :posts_in_year, ->(year) { where(created_at: year.beginning_of_year..year.end_of_year) },
           class_name: 'Post'
end

# Usage
user.posts_in_year(2024)
```

## Association Callbacks

```ruby
class User < ApplicationRecord
  has_many :posts,
           before_add: :check_limit,
           after_add: :send_notification,
           before_remove: :archive_post

  private

  def check_limit(post)
    raise "Post limit reached" if posts.count >= 100
  end

  def send_notification(post)
    NotificationJob.perform_later(self, post)
  end
end
```

## Best Practices

### Do
- Always specify `dependent:` option
- Use `inverse_of` for bidirectional associations
- Use counter caches for frequently accessed counts
- Eager load associations to prevent N+1
- Use `has_many :through` for join tables with attributes

### Don't
- Don't use `has_and_belongs_to_many` if join table needs attributes
- Don't forget to add database indexes for foreign keys
- Don't use callbacks for complex logic (use service objects)
