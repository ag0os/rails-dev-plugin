# Authentication Patterns — Detailed Reference

## Rails 8 Built-In Auth (Omakase)

### Complete Setup

```bash
bin/rails generate authentication
```

This generates:
- `User` model with `has_secure_password`
- `Session` model for tracking active sessions
- `SessionsController` for login/logout
- `Authentication` concern for `ApplicationController`
- `PasswordsController` for password resets
- `PasswordsMailer` for reset emails
- Migrations for users and sessions tables

### Session Model

```ruby
class Session < ApplicationRecord
  belongs_to :user

  # Track session metadata
  # Columns: user_id, user_agent, ip_address, created_at, updated_at
end
```

### Multiple Sessions Management

```ruby
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[new create]

  # Show all active sessions for current user
  def index
    @sessions = Current.user.sessions.order(created_at: :desc)
  end

  # Terminate a specific session (e.g., "sign out everywhere else")
  def destroy
    session = Current.user.sessions.find(params[:id])
    session.destroy
    redirect_to sessions_path, notice: "Session terminated."
  end

  # Terminate all other sessions
  def destroy_all
    Current.user.sessions.where.not(id: Current.session.id).destroy_all
    redirect_to sessions_path, notice: "All other sessions terminated."
  end
end
```

### Current Attributes

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end
```

### Rate Limiting (Rails 8)

```ruby
class SessionsController < ApplicationController
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

### Token Generation for Resets

```ruby
class User < ApplicationRecord
  has_secure_password

  # Password reset: expires in 15 minutes, invalidated when password changes
  generates_token_for :password_reset, expires_in: 15.minutes do
    password_salt&.last(10)
  end

  # Email confirmation: expires in 24 hours, invalidated when email changes
  generates_token_for :email_confirmation, expires_in: 24.hours do
    email_address
  end

  # Unsubscribe: never expires
  generates_token_for :unsubscribe
end

# Generate token
token = user.generate_token_for(:password_reset)

# Find user by token (returns nil if expired/invalid)
user = User.find_by_token_for(:password_reset, token)
# Or raise:
user = User.find_by_token_for!(:password_reset, token)
```

### Email Normalization

```ruby
class User < ApplicationRecord
  normalizes :email_address, with: -> { _1.strip.downcase }
  # User.find_by(email_address: " JANE@Example.COM ") works correctly
end
```

### allow_browser (Rails 8)

Block outdated browsers that don't support modern features:

```ruby
class ApplicationController < ActionController::Base
  allow_browser versions: :modern
  # Blocks browsers older than: Safari 17.2+, Chrome 120+, Firefox 121+, Opera 106+
end
```

## Devise (Service-Oriented)

### Installation and Configuration

```ruby
# Gemfile
gem "devise"
```

```ruby
# config/initializers/devise.rb (key settings)
Devise.setup do |config|
  config.mailer_sender = "noreply@example.com"
  config.password_length = 8..128
  config.reset_password_within = 6.hours
  config.sign_out_via = :delete
  config.navigational_formats = ["*/*", :html, :turbo_stream]
  config.responder.error_status = :unprocessable_entity
  config.responder.redirect_status = :see_other
end
```

### Devise Modules

| Module | Purpose | Adds |
|--------|---------|------|
| `:database_authenticatable` | Email + password login | `encrypted_password` column |
| `:registerable` | Signup/account management | Registration routes + controller |
| `:recoverable` | Password reset via email | `reset_password_token` columns |
| `:rememberable` | "Remember me" cookie | `remember_created_at` column |
| `:validatable` | Email/password validations | Format + length validations |
| `:confirmable` | Email confirmation | `confirmation_token` columns |
| `:lockable` | Lock after failed attempts | `locked_at`, `failed_attempts` columns |
| `:trackable` | Login tracking | `sign_in_count`, `last_sign_in_at` columns |
| `:timeoutable` | Session expiry | Timeout after inactivity |
| `:omniauthable` | OAuth (Google, GitHub, etc.) | OmniAuth integration |

### Custom Devise Controllers

```ruby
# config/routes.rb
devise_for :users, controllers: {
  registrations: "users/registrations",
  sessions: "users/sessions",
  confirmations: "users/confirmations"
}
```

```ruby
# app/controllers/users/registrations_controller.rb
class Users::RegistrationsController < Devise::RegistrationsController
  before_action :configure_sign_up_params, only: [:create]

  protected

  def configure_sign_up_params
    devise_parameter_sanitizer.permit(:sign_up, keys: [:name, :organization_name])
  end

  def after_sign_up_path_for(resource)
    onboarding_path
  end
end
```

### Devise with Turbo (Rails 7+)

Turbo requires specific status codes for redirects and errors:

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  config.responder.error_status = :unprocessable_entity
  config.responder.redirect_status = :see_other
end
```

```ruby
# If using custom failure app for Turbo
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

# In devise.rb initializer:
config.warden do |manager|
  manager.failure_app = TurboFailureApp
end
```

### Devise Test Helpers

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.include Devise::Test::IntegrationHelpers, type: :request
  config.include Devise::Test::IntegrationHelpers, type: :system
end

# In specs
RSpec.describe "Dashboard", type: :request do
  let(:user) { create(:user) }

  before { sign_in user }

  it "shows the dashboard" do
    get dashboard_path
    expect(response).to have_http_status(:ok)
  end
end
```

### OmniAuth Integration

```ruby
# Gemfile
gem "omniauth-google-oauth2"
gem "omniauth-rails_csrf_protection"

# config/initializers/devise.rb
config.omniauth :google_oauth2,
  Rails.application.credentials.dig(:google, :client_id),
  Rails.application.credentials.dig(:google, :client_secret)
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :omniauthable, omniauth_providers: [:google_oauth2]

  def self.from_omniauth(auth)
    find_or_create_by(provider: auth.provider, uid: auth.uid) do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0, 20]
      user.name = auth.info.name
    end
  end
end
```

## Security Best Practices

### Password Requirements

```ruby
# Omakase
class User < ApplicationRecord
  has_secure_password
  validates :password, length: { minimum: 8 }, if: :password_digest_changed?
end

# Devise handles this via config.password_length
```

### Secure Session Configuration

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: "_app_session",
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax,
  expire_after: 12.hours
```

### HTTPS Enforcement

```ruby
# config/environments/production.rb
config.force_ssl = true
config.ssl_options = { hsts: { subdomains: true, expires: 1.year } }
```

### Account Lockout (without Devise)

```ruby
class User < ApplicationRecord
  has_secure_password

  def lock_if_too_many_attempts!
    increment!(:failed_login_attempts)
    if failed_login_attempts >= 5
      update!(locked_at: Time.current)
    end
  end

  def locked?
    locked_at.present? && locked_at > 30.minutes.ago
  end

  def reset_failed_attempts!
    update!(failed_login_attempts: 0)
  end
end
```

## Directory Organization

### Omakase (built-in auth)

```
app/
  controllers/
    concerns/
      authentication.rb         # Authentication concern
    sessions_controller.rb      # Login/logout
    passwords_controller.rb     # Password reset
    registrations_controller.rb # Signup (if needed)
  models/
    current.rb                  # CurrentAttributes
    user.rb                     # has_secure_password
    session.rb                  # Session tracking
  mailers/
    passwords_mailer.rb         # Password reset email
  views/
    sessions/
      new.html.erb              # Login form
    passwords/
      new.html.erb              # Request reset form
      edit.html.erb             # Set new password form
```

### Service-oriented (Devise)

```
app/
  controllers/
    users/
      registrations_controller.rb  # Custom registration
      sessions_controller.rb       # Custom session handling
      confirmations_controller.rb  # Custom email confirmation
  models/
    user.rb                        # devise :database_authenticatable, ...
  views/
    devise/
      sessions/
        new.html.erb               # Login form
      registrations/
        new.html.erb               # Signup form
      passwords/
        new.html.erb               # Reset form
        edit.html.erb              # New password form
      confirmations/
        new.html.erb               # Resend confirmation
```
