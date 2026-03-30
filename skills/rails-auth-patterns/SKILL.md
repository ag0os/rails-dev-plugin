---
name: rails-auth-patterns
description: Authentication patterns for Rails applications. Automatically invoked when working with user login, signup, sessions, password resets, token authentication, has_secure_password, Devise, or auth configuration. Triggers on "authentication", "auth", "login", "signup", "session", "password", "has_secure_password", "Devise", "current_user", "sign_in", "sign_out", "remember me", "password reset", "email confirmation". NOT for authorization/permissions (use controller patterns for Pundit) or API token auth (use rails-api-patterns for JWT).
allowed-tools: Read, Grep, Glob
---

# Rails Authentication Patterns

Profile-aware authentication guidance. **Never suggest replacing one approach with another** unless explicitly asked.

See [patterns.md](patterns.md) for detailed code examples.

## Profile Detection

Check the project before recommending:

```
1. grep "devise" Gemfile → Devise (service-oriented)
2. grep "has_secure_password" app/models/ → Built-in auth (omakase)
3. Check for app/controllers/sessions_controller.rb → Custom auth
4. Rails 8+? → Built-in generator available
```

| Approach | Profile | Use When |
|----------|---------|----------|
| Rails 8 generator | Omakase (Rails 8+) | New projects, full control, no gem dependencies |
| `has_secure_password` (manual) | Omakase (pre-8) | Simple auth, full control |
| Devise | Service-oriented | Multi-feature auth (confirmable, lockable, omniauthable) |

## Rails 8 Built-In Auth

### What the Generator Creates

```bash
bin/rails generate authentication
# Creates: User model, Session model, SessionsController,
# Authentication concern, PasswordsController, PasswordsMailer, migrations
```

### Key Rails 8 Features (Non-Obvious)

**`generates_token_for` with automatic invalidation:**

```ruby
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  normalizes :email_address, with: -> { _1.strip.downcase }

  # Token auto-invalidates when password_salt changes (i.e., password changed)
  generates_token_for :password_reset, expires_in: 15.minutes do
    password_salt&.last(10)
  end

  # Token auto-invalidates when email changes
  generates_token_for :email_confirmation, expires_in: 24.hours do
    email_address
  end

  # Non-expiring token (e.g., unsubscribe links)
  generates_token_for :unsubscribe
end

# Generate: user.generate_token_for(:password_reset)
# Find:     User.find_by_token_for(:password_reset, token)  # nil if expired/invalid
# Find!:    User.find_by_token_for!(:password_reset, token) # raises if invalid
```

**CurrentAttributes pattern:**

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end
```

**Rate limiting (Rails 8 built-in):**

```ruby
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[new create]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> {
    redirect_to new_session_url, alert: "Try again later."
  }
end

class PasswordsController < ApplicationController
  rate_limit to: 5, within: 1.hour, only: :create, with: -> {
    redirect_to new_password_url, alert: "Too many reset requests."
  }
end
```

**Multiple session management:**

```ruby
# Allow users to see/terminate their active sessions
def destroy_all
  Current.user.sessions.where.not(id: Current.session.id).destroy_all
  redirect_to sessions_path, notice: "All other sessions terminated."
end
```

## Devise — Key Customization Points

Follow standard Devise setup/installation docs. The non-obvious parts:

**Turbo compatibility (Rails 7+) — required:**

```ruby
# config/initializers/devise.rb
config.responder.error_status = :unprocessable_entity
config.responder.redirect_status = :see_other
config.navigational_formats = ["*/*", :html, :turbo_stream]
```

**Custom failure app for Turbo (if redirects break):**

```ruby
class TurboFailureApp < Devise::FailureApp
  def respond
    if request_format == :turbo_stream
      redirect
    else
      super
    end
  end

  def skip_format?
    %w[html turbo_stream */*].include?(request_format.to_s)
  end
end

# config/initializers/devise.rb
config.warden do |manager|
  manager.failure_app = TurboFailureApp
end
```

See [patterns.md](patterns.md) for OmniAuth integration and custom controller patterns.

## Profile-Aware Test Helpers

**Omakase (Minitest):**

```ruby
class SessionsControllerTest < ActionDispatch::IntegrationTest
  test "login with valid credentials" do
    user = users(:jane)
    post session_url, params: { email_address: user.email_address, password: "password" }
    assert_redirected_to root_path
  end

  test "rate limiting after 10 attempts" do
    11.times { post session_url, params: { email_address: "x@x.com", password: "wrong" } }
    assert_redirected_to new_session_url
    follow_redirect!
    assert_match "Try again later", flash[:alert]
  end
end
```

**Service-oriented (RSpec + Devise):**

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.include Devise::Test::IntegrationHelpers, type: :request
  config.include Devise::Test::IntegrationHelpers, type: :system
end

# In specs — use sign_in helper, don't hit the login endpoint
before { sign_in user }
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| No rate limiting on login | Brute force attacks | `rate_limit` (Rails 8) or Rack::Attack |
| Password reset tokens without expiry | Token reuse attacks | `expires_in: 15.minutes` |
| Leaking user existence on reset | Enumeration attacks | Same response for valid/invalid emails |
| Session fixation | Hijacking | `reset_session` on login |
| Rolling your own token system | Crypto bugs | Use `generates_token_for` or Devise tokens |

## Output Format

When implementing auth, provide:
1. **Model** with auth configuration
2. **Controller(s)** for sessions, registrations, password resets
3. **Routes** configuration
4. **Test file** matching project profile (Minitest or RSpec)
5. **Security notes** (rate limiting, token expiry, session management)
