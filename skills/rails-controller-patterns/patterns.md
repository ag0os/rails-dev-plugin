# Controller Patterns for Rails

Detailed patterns beyond standard Rails controller conventions.

## Dedicated Resource Controller Implementation

Full example showing the resource controller pattern with a layered concern.

```ruby
# app/controllers/concerns/card_scoped.rb
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card, :set_board
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find_by!(number: params[:card_id])
    end

    def set_board
      @board = @card.board
    end

    def render_card_replacement
      render turbo_stream: turbo_stream.replace(
        [@card, :card_container],
        partial: "cards/container",
        method: :morph,
        locals: { card: @card.reload }
      )
    end
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    @card.close
    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end

  def destroy
    @card.reopen
    respond_to do |format|
      format.turbo_stream { render_card_replacement }
      format.json { head :no_content }
    end
  end
end
```

Layered concerns combine before_actions with shared helper methods, reducing duplication across multiple resource controllers that operate on the same parent.

## Authorization Patterns

Authorization is governed by an **orthogonal project fact**, not an architecture axis: which authorization library the project uses. Read it from the `project-conventions` fingerprint (Auth category — `pundit` / `cancancan` / `action_policy` / none). Match the project; never branch on a stack profile here.

### Model-Based Authorization (no authorization gem)

Use when no authorization gem is installed. Authorization logic lives in the domain model.

```ruby
# In controller
class BoardsController < ApplicationController
  before_action :set_board
  before_action :ensure_permission_to_admin_board, only: [:edit, :update, :destroy]

  private
    def ensure_permission_to_admin_board
      head :forbidden unless Current.user.can_administer_board?(@board)
    end
end

# In User model
class User < ApplicationRecord
  def can_administer_board?(board)
    admin? || board.creator == self
  end
end
```

### Pundit Policy Objects (authorization gem present)

Use when `pundit` (or a similar gem) is present. Follow existing conventions.

```ruby
class PostPolicy < ApplicationPolicy
  def update?
    record.user == user || user.admin?
  end
end

# In controller
class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    authorize @post
    # ...
  end
end
```

**Detection:** the `project-conventions` fingerprint's Auth category reports the authorization library. If a gem is present, use it. If none, use model-based auth.

## Streaming Responses

Use for large data exports to avoid memory bloat.

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

## Multi-Format Response with Turbo

```ruby
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

## Context Awareness Reference

| Pattern | Detection Method | Applicability |
|---------|------------------|---------------|
| **Resource Controllers** | Check `config/routes.rb` for custom actions | **Always** - replaces custom member actions |
| **Layered Concerns** | Check `app/controllers/concerns/` | **Always** - reduces duplication |
| **Model-Based Auth** | Check `Gemfile` for `pundit`, `cancancan` | **Conditional** - only if no auth gem present |
| **params.expect** | Check `Gemfile.lock` for Rails >= 8 | **Rails 8+** - fallback to `.require().permit()` |

### Detection Steps

**1. Check Rails Version**:
```bash
grep "rails (" Gemfile.lock
```

**2. Check Authorization Gems**:
```bash
grep -E "gem ['\"]pundit|cancancan" Gemfile
# If found: use existing gem. If absent: model-based auth.
```

**3. Check for Custom Actions to Refactor**:
```ruby
# In config/routes.rb, look for:
resources :posts do
  post :publish   # Candidate for Posts::PublicationsController
end
```

## Best Practices

### Do
- Delegate business logic to its Axis A home (`native`: models and concerns; `extracted`: service objects)
- Resolve Axis A via `rails-stack-profiles` before recommending where logic goes
- Match the project's authorization library from the `project-conventions` fingerprint
- Prefer dedicated resource controllers over custom actions
- Use layered concerns for shared setup logic

### Don't
- Suggest model-based auth when an authorization gem is installed
- Suggest a new auth gem when the project already has a working auth approach
- Use `params.expect` in Rails < 8
- Nest resources more than one level deep
