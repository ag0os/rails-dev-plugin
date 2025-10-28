# Quick Start Guide

This guide provides common workflows and examples for using the Rails Dev Plugin effectively.

## Installation

```shell
# In Claude Code, in any Rails project
/plugin marketplace add ag0os/rails-dev-plugin
/plugin install rails-dev-plugin@ag0os
```

Restart Claude Code after installation.

## Common Workflows

### 1. Building a New Feature

**Example: Adding a Blog System**

```
You: "I need to add a blog system with posts, comments, and tags"
```

**What happens**:
1. **rails-architect** agent plans the architecture
2. **rails-model** agent creates models with associations
3. **rails-controller** agent builds controllers
4. **rails-views** agent creates views and components
5. **rails-test** agent (proactively) adds comprehensive tests

**Claude's approach**:
```markdown
Let me help you build a blog system. I'll break this down into steps:

1. Architecture Planning (rails-architect)
   - Post has_many :comments
   - Post has_many :tags, through: :post_tags
   - Comment belongs_to :post

2. Model Creation (rails-model)
   - Generate migrations
   - Add validations
   - Define associations

3. Controller Setup (rails-controller)
   - PostsController with RESTful actions
   - CommentsController for nested resources
   - Authorization with Pundit

4. View Implementation (rails-views)
   - ViewComponents for post cards
   - Comment forms with Turbo
   - Tag filtering UI

5. Test Coverage (rails-test - automatic)
   - Model tests
   - Controller tests
   - System tests for user flows
```

### 2. Refactoring Fat Controllers

**Example: Extracting Complex Business Logic**

```
You: "My OrdersController is getting too complex with payment processing logic"
```

**What happens**:
1. **rails-architect** agent analyzes the controller
2. **rails-service** agent extracts logic into service objects
3. **rails-test** agent ensures test coverage is maintained
4. **Ruby Refactoring Expert** Skill identifies code smells

**Result**:
```ruby
# Before: Fat controller
class OrdersController < ApplicationController
  def create
    @order = current_user.orders.build(order_params)
    if @order.save
      charge = Stripe::Charge.create(...)
      @order.update(payment_id: charge.id)
      OrderMailer.confirmation(@order).deliver_later
      redirect_to @order
    else
      render :new
    end
  end
end

# After: Thin controller + service object
class OrdersController < ApplicationController
  def create
    result = Orders::CreateService.new(
      user: current_user,
      params: order_params
    ).call

    if result[:success]
      redirect_to result[:order]
    else
      @order = result[:order]
      render :new
    end
  end
end

# app/services/orders/create_service.rb
class Orders::CreateService
  # Clean, testable business logic
end
```

### 3. Adding Hotwire Interactivity

**Example: Real-time Updates**

```
You: "Add real-time updating to the comments section without page refresh"
```

**What happens**:
1. **rails-stimulus-turbo** agent implements Turbo Streams
2. **rails-views** agent updates view templates
3. **rails-controller** agent modifies controller responses

**Implementation**:
```ruby
# Controller
def create
  @comment = @post.comments.build(comment_params)

  if @comment.save
    respond_to do |format|
      format.turbo_stream  # New Turbo Stream response
      format.html { redirect_to @post }
    end
  end
end

# app/views/comments/create.turbo_stream.erb
<%= turbo_stream.append "comments", @comment %>
<%= turbo_stream.update "comment-form", partial: "comments/form" %>
```

### 4. Performance Optimization

**Example: Slow Index Page**

```
You: "The posts index page is loading slowly with 1000+ posts"
```

**What happens**:
1. **Rails Performance Analyzer** Skill detects N+1 queries
2. **rails-model** agent adds proper eager loading
3. **rails-controller** agent implements pagination
4. **rails-devops** agent suggests caching strategies

**Optimizations**:
```ruby
# Before: N+1 queries
@posts = Post.all  # Loads all posts
# In view: post.author.name triggers query per post

# After: Optimized
@posts = Post.includes(:author, :tags)
              .page(params[:page])
              .per(25)
# All data loaded efficiently with pagination
```

### 5. Security Audit

**Example: Pre-Production Security Review**

```
You: "Can you audit this application for security vulnerabilities?"
```

**What happens**:
1. **Rails Security Auditor** Skill scans for common vulnerabilities
2. Reports SQL injection risks, XSS issues, missing authorization
3. Provides specific fixes for each issue

**Sample findings**:
```markdown
## Security Audit Results

### Critical Vulnerabilities
1. SQL Injection in User search (app/controllers/users_controller.rb:15)
   - Fix: Use parameterized queries

2. Missing authorization check (app/controllers/orders_controller.rb:8)
   - Fix: Add `authorize @order`

3. XSS in comment rendering (app/views/comments/_comment.html.erb:5)
   - Fix: Use `sanitize` helper
```

### 6. Rails Version Upgrade

**Example: Upgrading from Rails 6.1 to 7.0**

```
You: "I need to upgrade from Rails 6.1 to Rails 7.0"
```

**What happens**:
1. **Rails Upgrade Assistant** Skill creates upgrade plan
2. Identifies breaking changes and required gem updates
3. Provides step-by-step migration guide

**Upgrade plan**:
```markdown
## Rails 6.1 → 7.0 Upgrade Plan

### Breaking Changes
1. Webpacker → Propshaft/Import Maps
2. ActiveStorage variants syntax changed
3. Turbolinks → Turbo

### Step-by-Step Process
1. Update Gemfile: `gem 'rails', '~> 7.0.0'`
2. Run: `bundle update rails`
3. Run: `rails app:update`
4. Address deprecations
5. Update gem dependencies
6. Test thoroughly

### Estimated Effort: 8-16 hours
```

### 7. Building GraphQL API

**Example: Adding GraphQL Endpoint**

```
You: "Add a GraphQL API for querying posts and comments"
```

**What happens**:
1. **rails-graphql** agent sets up GraphQL schema
2. Creates types, queries, and mutations
3. Implements N+1 query prevention
4. **rails-test** agent adds GraphQL specs

**Result**:
```ruby
# app/graphql/types/post_type.rb
class Types::PostType < Types::BaseObject
  field :id, ID, null: false
  field :title, String, null: false
  field :comments, [Types::CommentType], null: false

  def comments
    # Batch loaded to prevent N+1
    AssociationLoader.for(Post, :comments).load(object)
  end
end
```

### 8. DevOps Setup

**Example: CI/CD Pipeline**

```
You: "Set up GitHub Actions for automated testing and deployment"
```

**What happens**:
1. **rails-devops** agent creates workflow file
2. Configures test runners, linting, security scans
3. Sets up deployment to production
4. Adds environment variable management

**GitHub Actions workflow**:
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
      - name: Install dependencies
        run: bundle install
      - name: Run tests
        run: bundle exec rspec
      - name: Run security audit
        run: bundle exec brakeman
```

## Agent Decision Tree

Not sure which agent to use? See [agent-decision-tree.md](agent-decision-tree.md) for a visual guide.

## Tips for Best Results

### 1. Be Specific
```
❌ "Add authentication"
✅ "Add Devise authentication with email confirmation and password reset"
```

### 2. Mention the Context
```
❌ "Create a model"
✅ "Create an Order model with associations to User and LineItems"
```

### 3. Let Agents Work Proactively
The plugin agents will automatically invoke themselves when needed:
- After creating a model, the test agent will offer to create tests
- After writing business logic, the refactoring Skill may suggest improvements
- When planning features, the architect agent will provide guidance

### 4. Use Architecture Agent for Planning
Before implementing complex features:
```
You: "I want to add multi-currency support. What's the best approach?"
```
The rails-architect agent will help you think through the design before coding.

### 5. Leverage Skills for Analysis
Skills run automatically based on your questions:
- "Is this code maintainable?" → Ruby Refactoring Expert Skill
- "Are there security issues?" → Rails Security Auditor Skill
- "Why is this slow?" → Rails Performance Analyzer Skill

## Next Steps

- **Explore agent capabilities**: Try different prompts to see how agents respond
- **Review agent documentation**: Check individual agent markdown files for details
- **Customize for your team**: Fork and adjust agents to match your conventions
- **Report issues**: Found a bug or have a suggestion? Open an issue on GitHub

## Getting Help

- **Plugin issues**: https://github.com/ag0os/rails-dev-plugin/issues
- **Claude Code docs**: https://docs.claude.com/en/docs/claude-code
- **Rails guides**: https://guides.rubyonrails.org

Happy coding! 🚀
