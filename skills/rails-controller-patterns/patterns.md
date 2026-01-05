# Controller Patterns for Rails

Detailed patterns for Rails controllers.

## Response Handling

### Multiple Formats

```ruby
def show
  @post = Post.find(params[:id])

  respond_to do |format|
    format.html
    format.json { render json: @post }
    format.turbo_stream
  end
end

def create
  @post = current_user.posts.build(post_params)

  respond_to do |format|
    if @post.save
      format.html { redirect_to @post, notice: 'Created!' }
      format.json { render json: @post, status: :created }
      format.turbo_stream
    else
      format.html { render :new, status: :unprocessable_entity }
      format.json { render json: @post.errors, status: :unprocessable_entity }
      format.turbo_stream { render :form_errors, status: :unprocessable_entity }
    end
  end
end
```

### JSON API

```ruby
class Api::V1::PostsController < Api::V1::BaseController
  def index
    @posts = Post.includes(:user).page(params[:page])
    render json: @posts, each_serializer: PostSerializer
  end

  def show
    @post = Post.find(params[:id])
    render json: @post, serializer: PostSerializer
  end

  def create
    @post = current_user.posts.build(post_params)

    if @post.save
      render json: @post, status: :created
    else
      render json: { errors: @post.errors }, status: :unprocessable_entity
    end
  end
end
```

## Before Actions

### Common Patterns

```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :authorize_post!, only: [:edit, :update, :destroy]

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def authorize_post!
    redirect_to posts_path, alert: 'Not authorized' unless @post.user == current_user
  end
end
```

### Skipping Filters

```ruby
class SessionsController < ApplicationController
  skip_before_action :authenticate_user!, only: [:new, :create]
end
```

## Error Handling

### Rescue From

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from Pundit::NotAuthorizedError, with: :forbidden

  private

  def not_found
    respond_to do |format|
      format.html { render 'errors/not_found', status: :not_found }
      format.json { render json: { error: 'Not found' }, status: :not_found }
    end
  end

  def forbidden
    respond_to do |format|
      format.html { redirect_to root_path, alert: 'Access denied' }
      format.json { render json: { error: 'Forbidden' }, status: :forbidden }
    end
  end
end
```

### Manual Error Handling

```ruby
def create
  @post = current_user.posts.build(post_params)

  if @post.save
    redirect_to @post
  else
    flash.now[:alert] = 'Could not create post'
    render :new, status: :unprocessable_entity
  end
rescue ExternalServiceError => e
  flash[:alert] = 'Service temporarily unavailable'
  redirect_to posts_path
end
```

## Routing

### RESTful Resources

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts do
    resources :comments, shallow: true
    member do
      post :publish
      delete :archive
    end
    collection do
      get :search
      get :drafts
    end
  end
end
```

### Nested Resources

```ruby
resources :users do
  resources :posts, only: [:index, :new, :create]
end

# Generates:
# GET    /users/:user_id/posts
# GET    /users/:user_id/posts/new
# POST   /users/:user_id/posts
```

### Namespaced Routes

```ruby
namespace :admin do
  resources :users
  resources :posts
end

namespace :api do
  namespace :v1 do
    resources :posts, only: [:index, :show, :create]
  end
end
```

### Custom Routes

```ruby
# Singular resource (no :id)
resource :profile, only: [:show, :edit, :update]

# Non-resourceful routes
get 'about', to: 'pages#about'
post 'login', to: 'sessions#create'

# Root route
root 'home#index'
```

## Strong Parameters Patterns

### Basic Usage

```ruby
def post_params
  params.expect(post: [:title, :content, :published])
end
```

### Nested Attributes

```ruby
def user_params
  params.expect(user: [
    :name,
    :email,
    profile_attributes: [:bio, :website, :id, :_destroy],
    addresses_attributes: [:street, :city, :id, :_destroy]
  ])
end
```

### Dynamic/Hash Attributes

```ruby
def product_params
  params.expect(product: [
    :name,
    :price,
    metadata: {}  # Allow any hash
  ])
end
```

### Array Parameters

```ruby
def post_params
  params.expect(post: [
    :title,
    tag_ids: [],           # Array of IDs
    categories: []         # Array of values
  ])
end
```

## Concerns

### Shared Behavior

```ruby
# app/controllers/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  included do
    helper_method :search_results
  end

  def search
    @results = search_results
    render :search_results
  end

  private

  def search_results
    model_class.search(params[:q]).page(params[:page])
  end

  def model_class
    controller_name.classify.constantize
  end
end

# Usage
class PostsController < ApplicationController
  include Searchable
end
```

## Pagination Pattern

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.published
                 .includes(:user, :category)
                 .order(created_at: :desc)
                 .page(params[:page])
                 .per(25)
  end
end
```

## Streaming Responses

```ruby
class ReportsController < ApplicationController
  def export
    respond_to do |format|
      format.csv do
        headers['Content-Disposition'] = 'attachment; filename="report.csv"'
        headers['Content-Type'] = 'text/csv'

        self.response_body = Enumerator.new do |yielder|
          yielder << CSV.generate_line(['ID', 'Name', 'Email'])
          User.find_each do |user|
            yielder << CSV.generate_line([user.id, user.name, user.email])
          end
        end
      end
    end
  end
end
```

## Flash Messages

```ruby
def create
  @post = current_user.posts.build(post_params)

  if @post.save
    redirect_to @post, notice: 'Post created successfully.'
  else
    flash.now[:alert] = 'Could not create post.'
    render :new, status: :unprocessable_entity
  end
end

def destroy
  @post.destroy
  redirect_to posts_path, notice: 'Post deleted.', status: :see_other
end
```

## Status Codes

| Status | Code | Use When |
|--------|------|----------|
| `:ok` | 200 | Successful GET/PUT/PATCH |
| `:created` | 201 | Successful POST |
| `:no_content` | 204 | Successful DELETE (no body) |
| `:see_other` | 303 | Redirect after DELETE |
| `:bad_request` | 400 | Invalid request format |
| `:unauthorized` | 401 | Authentication required |
| `:forbidden` | 403 | Permission denied |
| `:not_found` | 404 | Resource not found |
| `:unprocessable_entity` | 422 | Validation errors |
| `:internal_server_error` | 500 | Server error |

## API Controller Base

```ruby
class Api::V1::BaseController < ActionController::API
  include ActionController::HttpAuthentication::Token::ControllerMethods

  before_action :authenticate_token!

  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity

  private

  def authenticate_token!
    authenticate_or_request_with_http_token do |token, _options|
      @current_user = User.find_by(api_token: token)
    end
  end

  def current_user
    @current_user
  end

  def not_found
    render json: { error: 'Not found' }, status: :not_found
  end

  def unprocessable_entity(exception)
    render json: { errors: exception.record.errors }, status: :unprocessable_entity
  end
end
```

## Best Practices

### Do
- Keep controllers thin
- Use before_actions for repetitive setup
- Return appropriate status codes
- Use strong parameters
- Delegate business logic to models/services

### Don't
- Put business logic in controllers
- Use instance variables excessively
- Skip validation status codes
- Nest resources more than one level deep
- Create non-RESTful actions without good reason
