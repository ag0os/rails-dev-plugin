# Authentication Patterns — Detailed Reference

## Devise Turbo Integration (Complete)

Turbo requires specific response codes. This is the most common source of Devise + Rails 7+ issues.

**Required initializer settings:**

```ruby
# config/initializers/devise.rb
Devise.setup do |config|
  config.responder.error_status = :unprocessable_entity
  config.responder.redirect_status = :see_other
  config.navigational_formats = ["*/*", :html, :turbo_stream]
end
```

## Devise Custom Controllers

When you need to add fields or change redirect behavior:

```ruby
# config/routes.rb
devise_for :users, controllers: {
  registrations: "users/registrations",
  sessions: "users/sessions"
}
```

```ruby
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

## OmniAuth Integration with Devise

```ruby
# Gemfile
gem "omniauth-google-oauth2"
gem "omniauth-rails_csrf_protection"  # Required for CSRF protection
```

```ruby
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

## Account Lockout Without Devise

When using built-in auth and you need brute-force protection beyond rate limiting:

```ruby
class User < ApplicationRecord
  has_secure_password

  def lock_if_too_many_attempts!
    increment!(:failed_login_attempts)
    update!(locked_at: Time.current) if failed_login_attempts >= 5
  end

  def locked?
    locked_at.present? && locked_at > 30.minutes.ago
  end

  def reset_failed_attempts!
    update!(failed_login_attempts: 0)
  end
end
```

## Security Checklist

Apply these regardless of which auth approach is used:

```ruby
# Force HTTPS in production
# config/environments/production.rb
config.force_ssl = true

# Session security
Rails.application.config.session_store :cookie_store,
  key: "_app_session",
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax,
  expire_after: 12.hours
```
