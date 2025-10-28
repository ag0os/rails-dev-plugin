---
name: Rails Upgrade Assistant
description: Automatically invoked when upgrading Rails versions or dealing with deprecations. Triggers on mentions of "upgrade Rails", "Rails version", "deprecation", "migration guide", "breaking change", "upgrade to Rails", "compatibility", "Rails 6 to 7", "Rails 7 to 8", "deprecated feature", "upgrade path", "version bump". Provides guidance for Rails version upgrades and handling deprecations.
allowed-tools: Read, Grep, Glob, Bash
---

# Rails Upgrade Assistant

Systematic guidance for upgrading Rails applications to newer versions.

## When to Use This Skill

- Planning a Rails version upgrade
- Dealing with deprecation warnings
- Understanding breaking changes between Rails versions
- Migrating from older Rails patterns to modern conventions
- Troubleshooting upgrade-related issues
- Reviewing compatibility of gems with new Rails versions

## Upgrade Methodology

### 1. Pre-Upgrade Assessment

**Check current Rails version**:
```bash
bundle exec rails version
grep "gem 'rails'" Gemfile
```

**Review deprecation warnings**:
```bash
# Enable all deprecations in development
# config/environments/development.rb
config.active_support.deprecation = :log

# Run test suite and look for deprecations
bundle exec rspec
bundle exec rails test
```

**Audit gem compatibility**:
```bash
# Check which gems need updates
bundle outdated

# Review Gemfile for Rails version constraints
grep rails Gemfile
```

### 2. Upgrade Strategy

**Recommended approach**:
1. Upgrade one minor version at a time (6.0 → 6.1 → 7.0 → 7.1)
2. Read official upgrade guides for each version
3. Update gems after Rails core
4. Run tests at each step
5. Address deprecations before moving to next version

**Create upgrade branch**:
```bash
git checkout -b rails-upgrade
```

### 3. Common Upgrade Patterns

## Rails 6.0 to 6.1

**Major changes**:
- `ActiveRecord::Base.connection_config` removed
- `before_action` callbacks must explicitly return `false` to halt
- Zeitwerk autoloader becomes default
- Per-environment credentials

**Deprecations to address**:
```ruby
# ❌ Deprecated in 6.1
User.connection.structure_dump

# ✅ Use schema dumper
ActiveRecord::Tasks::DatabaseTasks.structure_dump(config)
```

## Rails 6.1 to 7.0

**Major breaking changes**:

1. **Active Storage variant changes**:
```ruby
# ❌ Rails 6.1 and earlier
user.avatar.variant(resize: "100x100")

# ✅ Rails 7.0+
user.avatar.variant(resize_to_limit: [100, 100])
```

2. **Turbolinks replaced by Turbo**:
```ruby
# Remove from Gemfile:
gem 'turbolinks'

# Add Hotwire:
gem 'hotwire-rails'
```

3. **Webpacker removed (optional)**:
```bash
# New default: Import maps (Propshaft)
# Or use jsbundling-rails with esbuild/webpack
```

4. **`config.autoloader` must be `:zeitwerk`**:
```ruby
# config/application.rb
config.autoloader = :zeitwerk  # Classic autoloader removed
```

5. **`ActiveRecord::Base.connection.structure_dump` removed**:
Use `schema.rb` or `structure.sql` via `db:schema:dump`

## Rails 7.0 to 7.1

**Major changes**:

1. **Deprecated `Rails.application.config.action_view.raise_on_missing_translations`**:
```ruby
# config/environments/development.rb
# ❌ Deprecated
config.action_view.raise_on_missing_translations = true

# ✅ Use I18n config
config.i18n.raise_on_missing_translations = true
```

2. **New default behavior for associations**:
```ruby
# Inverse associations are now required by default
class Post < ApplicationRecord
  has_many :comments, inverse_of: :post
end

class Comment < ApplicationRecord
  belongs_to :post, inverse_of: :comments
end
```

3. **`ActiveSupport::Deprecation` behavior changes**:
Warnings now raise errors in test environment by default

4. **Enumerable improvements**:
```ruby
# ✅ New method available
[1, 2, 3].in_order_of(:id, [3, 1, 2])
```

## Rails 7.1 to 7.2/8.0 (Future)

**Upcoming changes** (check official guides):
- Further Hotwire integration
- Performance improvements
- Security enhancements
- Possible removal of older deprecated features

### 4. Gem Compatibility

**Check gem compatibility matrix**:
```bash
# See which gems are compatible with target Rails version
bundle update --conservative rails

# Check specific gem compatibility
bundle info <gem_name>
```

**Common gem updates needed**:
```ruby
# Gemfile
gem 'rails', '~> 7.1.0'

# Usually need updates:
gem 'devise'              # Authentication
gem 'pundit'              # Authorization
gem 'paperclip'           # → Migrate to ActiveStorage
gem 'factory_bot_rails'   # Test factories
gem 'rspec-rails'         # Testing
gem 'capybara'            # System tests
```

### 5. Configuration Updates

**Generate and compare new configs**:
```bash
# Generate new Rails app with target version for reference
rails new reference_app --skip-test --skip-bundle

# Compare configuration files
diff config/application.rb reference_app/config/application.rb
```

**Update credentials system** (Rails 6+):
```bash
# Migrate from secrets.yml to credentials
rails credentials:edit

# Per-environment credentials (Rails 6.1+)
rails credentials:edit --environment production
```

### 6. Code Pattern Migrations

**Update controller patterns**:
```ruby
# ❌ Old pattern (before Rails 5)
before_filter :authenticate_user!

# ✅ Modern pattern
before_action :authenticate_user!
```

**Update query patterns**:
```ruby
# ❌ Deprecated finder methods
User.find_all_by_role('admin')

# ✅ Modern ActiveRecord
User.where(role: 'admin')
```

**Update validation patterns**:
```ruby
# ❌ Old validation syntax
validates_presence_of :name
validates_uniqueness_of :email

# ✅ Modern syntax
validates :name, presence: true
validates :email, uniqueness: true
```

## Upgrade Checklist

### Before Upgrading
- [ ] Full test suite is passing
- [ ] Database is backed up
- [ ] All deprecation warnings are addressed
- [ ] Dependencies are documented
- [ ] Upgrade path is planned (version by version)

### During Upgrade
- [ ] Update Rails version in Gemfile
- [ ] Run `bundle update rails`
- [ ] Run `rails app:update` (review each change carefully)
- [ ] Update database configurations
- [ ] Update environment configurations
- [ ] Address deprecation warnings
- [ ] Run test suite after each change

### After Upgrading
- [ ] All tests passing
- [ ] Manual testing in development
- [ ] Staging environment deployment
- [ ] Performance testing
- [ ] Security audit
- [ ] Production deployment plan

## Running the Upgrade

**Step-by-step process**:

```bash
# 1. Update Rails version in Gemfile
# Gemfile
gem 'rails', '~> 7.1.0'

# 2. Update dependencies
bundle update rails

# 3. Run Rails upgrade task
rails app:update

# Review each file conflict carefully:
# - Keep your changes if they're custom
# - Adopt new Rails defaults when appropriate
# - Document any skipped updates

# 4. Update configuration
# Review and update config files manually

# 5. Run tests
bundle exec rspec
bundle exec rails test

# 6. Fix deprecation warnings
# Check logs for deprecation messages

# 7. Update database (if needed)
rails db:migrate

# 8. Test in development
rails server
# Manually test critical user flows
```

## Common Issues and Solutions

### Issue: Missing Asset Pipeline
```bash
# If using Sprockets
gem 'sprockets-rails'

# If using Webpacker (Rails 6)
gem 'webpacker'

# If using Propshaft (Rails 7+)
gem 'propshaft'
```

### Issue: Autoloader Errors
```ruby
# config/application.rb
config.autoloader = :zeitwerk
config.eager_load_paths << Rails.root.join('lib')

# Ensure file names match constant names:
# app/models/user.rb → class User
# app/models/order_item.rb → class OrderItem
```

### Issue: ActiveRecord Query Deprecations
```ruby
# ❌ Deprecated
User.find_by_email('test@example.com')

# ✅ Use hash syntax
User.find_by(email: 'test@example.com')
```

### Issue: Rack Middleware Changes
```ruby
# config/application.rb
# May need to update middleware stack
config.middleware.use SomeMiddleware
```

## Testing Strategy

**Comprehensive testing approach**:

1. **Unit tests**: Ensure all model and service tests pass
2. **Integration tests**: Test controller actions and API endpoints
3. **System tests**: Run Capybara/System tests for critical flows
4. **Performance tests**: Benchmark key operations
5. **Manual testing**: Test critical user journeys in browser

**Test coverage check**:
```bash
# Run with coverage
bundle exec rspec --tag ~slow

# Check deprecation warnings
DEPRECATION_WARNINGS=1 bundle exec rspec
```

## Rollback Plan

**Always have a rollback strategy**:

```bash
# 1. Keep previous Gemfile.lock
cp Gemfile.lock Gemfile.lock.backup

# 2. Tag before upgrade
git tag -a pre-rails-upgrade -m "Before Rails upgrade"

# 3. If rollback needed:
git checkout Gemfile Gemfile.lock
bundle install
rails db:rollback  # If migrations were run
```

## Version-Specific Resources

**Official upgrade guides**:
- Rails 6.0: https://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#upgrading-from-rails-5-2-to-rails-6-0
- Rails 6.1: https://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#upgrading-from-rails-6-0-to-rails-6-1
- Rails 7.0: https://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#upgrading-from-rails-6-1-to-rails-7-0
- Rails 7.1: https://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#upgrading-from-rails-7-0-to-rails-7-1

**Use MCP tools to fetch latest upgrade guides when available**.

## Output Format

Provide upgrade guidance in this structure:

```markdown
## Rails Upgrade Analysis

### Current State
- Rails version: X.X.X
- Ruby version: X.X.X
- Target Rails version: X.X.X

### Recommended Upgrade Path
1. Version X.X → X.X
2. Version X.X → X.X
3. Version X.X → X.X

### Breaking Changes to Address
1. [Change description]
   - Location: [files affected]
   - Action required: [specific fixes]

### Gem Updates Required
- [gem name]: current version → target version

### Configuration Changes
...

### Estimated Effort
- Time estimate: [hours/days]
- Risk level: [Low/Medium/High]
- Testing required: [scope]

### Step-by-Step Plan
...
```

Remember: Patience is key. Upgrade incrementally, test thoroughly, and don't skip versions unless absolutely necessary.
