---
name: rails-auth-patterns
description: Authentication patterns for Rails applications. Automatically invoked when working with user login, signup, sessions, password resets, token authentication, has_secure_password, Devise, or auth configuration. Triggers on "authentication", "auth", "login", "signup", "session", "password", "has_secure_password", "Devise", "current_user", "sign_in", "sign_out", "remember me", "password reset", "email confirmation". NOT for authorization/permissions (use controller patterns for Pundit) or API token auth (use rails-api-patterns for JWT).
allowed-tools: Read, Grep, Glob
---

# Rails Authentication Patterns

Analyze and recommend authentication patterns for Rails applications. Profile-aware: Rails 8 built-in auth (omakase) vs Devise (service-oriented).

See [patterns.md](patterns.md) for detailed code examples.

## Quick Reference

| Approach | Profile | Use When |
|----------|---------|----------|
| `has_secure_password` + sessions | Omakase | Simple auth needs, full control, no gem dependencies |
| Devise | Service-oriented | Multi-feature auth (confirmable, lockable, omniauthable) |
| Rails 8 generator (`bin/rails generate authentication`) | Omakase (Rails 8+) | Quickest start with built-in auth |

## Profile Detection

Check the project before recommending an approach:

```
1. grep "devise" Gemfile → Service-oriented auth
2. grep "has_secure_password" app/models/ → Omakase auth
3. Check for app/controllers/sessions_controller.rb → Custom auth
4. Rails 8+? → Built-in generator available
```

**Never suggest replacing Devise with built-in auth** (or vice versa) unless the user explicitly asks.

## Rails 8 Built-In Auth (Omakase)

### Generator

```bash
bin/rails generate authentication
# Creates: User model, Session model, SessionsController,
# Authentication concern, password reset mailer, migrations
```

### User Model

```ruby
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  normalizes :email_address, with: -> { _1.strip.downcase }

  generates_token_for :password_reset, expires_in: 15.minutes do
    password_salt&.last(10)
  end

  generates_token_for :email_confirmation, expires_in: 24.hours do
    email_address
  end
end
```

See [patterns.md](patterns.md) for the full Authentication concern, password reset, session management, and security hardening patterns.

### Sessions Controller

```ruby
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[new create]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> {
    redirect_to new_session_url, alert: "Try again later."
  }

  def new; end

  def create
    if user = User.authenticate_by(email_address: params[:email_address], password: params[:password])
      start_new_session_for user
      redirect_to after_authentication_url
    else
      redirect_to new_session_path, alert: "Invalid email or password."
    end
  end

  def destroy
    terminate_session
    redirect_to new_session_path
  end
end
```

## Devise (Service-Oriented)

```ruby
# Gemfile
gem "devise"

# bin/rails generate devise:install
# bin/rails generate devise User
```

```ruby
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :lockable, :trackable
end
```

```ruby
# Routes
devise_for :users, controllers: {
  registrations: "users/registrations",
  sessions: "users/sessions"
}
```

See [patterns.md](patterns.md) for Devise modules reference, custom controllers, Turbo integration, OmniAuth, and test helpers.

## Testing Auth

**Omakase — Minitest:**

```ruby
class SessionsControllerTest < ActionDispatch::IntegrationTest
  test "login with valid credentials" do
    user = users(:jane)
    post session_url, params: { email_address: user.email_address, password: "password" }
    assert_redirected_to root_path
    assert cookies[:session_id].present?
  end

  test "login with invalid credentials" do
    post session_url, params: { email_address: "wrong@example.com", password: "bad" }
    assert_redirected_to new_session_path
  end
end
```

**Service-oriented — RSpec + Devise helpers:**

```ruby
RSpec.describe "Authentication", type: :request do
  let(:user) { create(:user) }

  describe "POST /users/sign_in" do
    it "signs in with valid credentials" do
      post user_session_path, params: { user: { email: user.email, password: user.password } }
      expect(response).to redirect_to(root_path)
    end
  end
end

# In rails_helper.rb
RSpec.configure do |config|
  config.include Devise::Test::IntegrationHelpers, type: :request
end
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Storing passwords in plain text | Security catastrophe | Always use `has_secure_password` or Devise |
| No rate limiting on login | Brute force attacks | Use `rate_limit` (Rails 8) or Rack::Attack |
| Password reset tokens that don't expire | Token reuse attacks | Set expiry (`expires_in: 15.minutes`) |
| Session fixation | Hijacking | Rotate session on login (`reset_session`) |
| No CSRF protection | Cross-site request forgery | Keep `protect_from_forgery` enabled |
| Leaking user existence | Enumeration attacks | Same response for valid/invalid emails on reset |

## Output Format

When implementing auth, provide:
1. **Model** with auth configuration
2. **Controller(s)** for sessions, registrations, password resets
3. **Routes** configuration
4. **Views** for login/signup forms
5. **Test file** matching project profile
6. **Security notes** (rate limiting, CSRF, token expiry)
