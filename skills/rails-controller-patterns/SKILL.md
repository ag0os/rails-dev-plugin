---
name: rails-controller-patterns
description: Analyzes and recommends Rails controller patterns including RESTful design, strong parameters, before_actions, response handling, and routing. Use when building controllers, defining routes, handling params, or managing request/response flow. NOT for model validations, service object internals, view templates, or background job logic.
allowed-tools: Read, Grep, Glob
---

# Rails Controller Patterns

Analyze and recommend patterns for well-structured, thin Rails controllers.

## Quick Reference

| Action | HTTP Verb | Path | Purpose |
|--------|-----------|------|---------|
| index | GET | /posts | List all |
| show | GET | /posts/:id | Show one |
| new | GET | /posts/new | New form |
| create | POST | /posts | Create |
| edit | GET | /posts/:id/edit | Edit form |
| update | PATCH/PUT | /posts/:id | Update |
| destroy | DELETE | /posts/:id | Delete |

## Supporting Documentation

- [patterns.md](patterns.md) - Detailed controller patterns and examples

## Core Principles

1. **Thin controllers**: Handle HTTP concerns only; delegate business logic to models/services
2. **RESTful by default**: Stick to 7 standard actions; create new controllers for custom actions
3. **Strong parameters always**: Never trust user input; use `params.expect` (Rails 8+)
4. **Consistent responses**: `redirect_to` after success, `render :action, status:` on failure
5. **One resource per controller**: Avoid multi-resource controllers

## Controller Structure Template

```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_post, only: %i[show edit update destroy]
  before_action :authorize_post, only: %i[edit update destroy]

  def index
    @posts = Post.recent.page(params[:page])
  end

  def show; end

  def create
    @post = current_user.posts.build(post_params)
    if @post.save
      redirect_to @post, notice: "Post created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    if @post.update(post_params)
      redirect_to @post, notice: "Post updated."
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @post.destroy
    redirect_to posts_path, notice: "Post deleted.", status: :see_other
  end

  private

  def set_post
    @post = current_user.posts.find(params[:id])
  end

  def authorize_post
    redirect_to posts_path, alert: "Not authorized." unless @post.user == current_user
  end

  def post_params
    params.expect(post: [:title, :content, :published, tags: []])
  end
end
```

## Strong Parameters (Rails 8+)

```ruby
# Basic - params.expect replaces require + permit
params.expect(post: [:title, :content])

# Arrays
params.expect(post: [:title, tags: []])

# Nested attributes
params.expect(user: [:name, :email, profile_attributes: [:bio, :avatar]])

# Legacy syntax (still works)
params.require(:post).permit(:title, :content)
```

## Error Handling with rescue_from

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActionPolicy::Unauthorized, with: :forbidden

  private

  def not_found
    respond_to do |format|
      format.html { render "errors/not_found", status: :not_found }
      format.json { render json: { error: "Not found" }, status: :not_found }
    end
  end

  def forbidden
    respond_to do |format|
      format.html { redirect_back fallback_location: root_path, alert: "Not authorized." }
      format.json { render json: { error: "Forbidden" }, status: :forbidden }
    end
  end
end
```

## Response Handling

```ruby
# Multi-format
respond_to do |format|
  format.html { redirect_to @post }
  format.turbo_stream { render turbo_stream: turbo_stream.replace(@post) }
  format.json { render json: @post, status: :ok }
end

# API controller
class Api::V1::PostsController < ActionController::API
  def index
    posts = Post.recent.limit(20)
    render json: posts, status: :ok
  end
end
```

## Routing Patterns

```ruby
resources :posts do
  resources :comments, only: %i[create destroy], shallow: true
  member { post :publish }
  collection { get :drafts }
end

# Namespace for API versioning
namespace :api do
  namespace :v1 do
    resources :posts, only: %i[index show create]
  end
end
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Fat controllers (50+ line actions) | Hard to test and maintain | Extract to service objects |
| Business logic in controllers | Violates SRP | Move to models or services |
| `params.permit!` | Allows all params (mass assignment) | Use `params.expect` with explicit fields |
| Deeply nested routes (>1 level) | Confusing URLs and helpers | Use `shallow: true` or flat routes |
| Skipping authentication filters | Security vulnerability | Apply `before_action` broadly, skip selectively |
| Rendering in non-standard status | Confusing HTTP semantics | Use correct status codes (422, 404, 403) |
| Multiple `respond_to` blocks | Code duplication | Use `respond_to` once or separate API controllers |

## Output Format

When analyzing or creating controllers, provide:
1. **Controller file** with proper structure and thin actions
2. **Route entries** for `config/routes.rb`
3. **Before actions** for auth, resource loading
4. **Strong params** method with explicit field list
5. **Error handling** strategy (rescue_from or inline)

## Error Handling Checklist

- Return `status: :unprocessable_entity` (422) for validation failures
- Return `status: :not_found` (404) for missing records
- Return `status: :forbidden` (403) for authorization failures
- Return `status: :see_other` (303) for `redirect_to` after DELETE
- Use `rescue_from` in ApplicationController for consistent error responses
- Log unexpected errors with context before re-raising
