# Rails DevOps Patterns -- Detailed Reference

## Docker Configuration

### Production Dockerfile (Multi-stage)

```dockerfile
# syntax=docker/dockerfile:1
FROM ruby:3.2-alpine AS base
RUN apk add --no-cache postgresql-dev tzdata gcompat
WORKDIR /app
ENV RAILS_ENV=production \
    BUNDLE_DEPLOYMENT=true \
    BUNDLE_WITHOUT="development:test"

FROM base AS build
RUN apk add --no-cache build-base git nodejs yarn
COPY Gemfile Gemfile.lock ./
RUN bundle install --jobs 4 --retry 3
COPY package.json yarn.lock ./
RUN yarn install --production --frozen-lockfile
COPY . .
RUN SECRET_KEY_BASE=precompile_placeholder bundle exec rails assets:precompile

FROM base AS runtime
COPY --from=build /usr/local/bundle /usr/local/bundle
COPY --from=build /app /app
RUN addgroup -S rails && adduser -S rails -G rails && \
    chown -R rails:rails /app
USER rails
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/up || exit 1
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

### Docker Compose (Development)

```yaml
version: "3.8"

services:
  web:
    build: .
    command: bundle exec rails server -b 0.0.0.0
    volumes:
      - .:/app
      - bundle_cache:/usr/local/bundle
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    environment:
      DATABASE_URL: postgres://postgres:password@db:5432/myapp_development
      REDIS_URL: redis://redis:6379/0

  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

  sidekiq:
    build: .
    command: bundle exec sidekiq
    depends_on:
      - db
      - redis
    environment:
      DATABASE_URL: postgres://postgres:password@db:5432/myapp_development
      REDIS_URL: redis://redis:6379/0

volumes:
  postgres_data:
  redis_data:
  bundle_cache:
```

## CI/CD -- GitHub Actions

### Full CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.2"
          bundler-cache: true
      - run: bundle exec rubocop --parallel
      - run: bundle exec brakeman -q --no-pager

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.2"
          bundler-cache: true
      - name: Setup database
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
          RAILS_ENV: test
        run: |
          bundle exec rails db:create db:schema:load
      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
          REDIS_URL: redis://localhost:6379/0
          RAILS_ENV: test
        run: bundle exec rspec --format progress
      - name: Upload coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.2"
          bundler-cache: true
      - run: bundle exec bundler-audit check --update
      - run: bundle exec brakeman -q --no-pager
```

## Puma Configuration

```ruby
# config/puma.rb
max_threads_count = ENV.fetch("RAILS_MAX_THREADS", 5).to_i
min_threads_count = ENV.fetch("RAILS_MIN_THREADS", max_threads_count).to_i
threads min_threads_count, max_threads_count

port ENV.fetch("PORT", 3000)
environment ENV.fetch("RAILS_ENV", "development")
pidfile ENV.fetch("PIDFILE", "tmp/pids/server.pid")

if ENV["RAILS_ENV"] == "production"
  workers ENV.fetch("WEB_CONCURRENCY", 2).to_i
  preload_app!

  before_fork do
    ActiveRecord::Base.connection_pool.disconnect! if defined?(ActiveRecord)
  end

  on_worker_boot do
    ActiveRecord::Base.establish_connection if defined?(ActiveRecord)
  end
end
```

## Monitoring and Logging

### Structured JSON Logging

```ruby
# config/environments/production.rb
config.logger = ActiveSupport::TaggedLogging.new(
  Logger.new(STDOUT).tap do |logger|
    logger.formatter = proc do |severity, time, _progname, msg|
      {
        severity: severity,
        time: time.iso8601(3),
        msg: msg,
        host: Socket.gethostname,
        pid: Process.pid,
        tid: Thread.current.object_id
      }.to_json + "\n"
    end
  end
)
config.log_tags = [:request_id]
config.log_level = ENV.fetch("LOG_LEVEL", "info").to_sym
```

### Lograge (Compact Request Logs)

```ruby
# config/initializers/lograge.rb
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Json.new
  config.lograge.custom_payload do |controller|
    {
      user_id: controller.current_user&.id,
      request_id: controller.request.request_id
    }
  end
end
```

### Prometheus Metrics

```ruby
# config/initializers/prometheus.rb
if Rails.env.production?
  require "prometheus/client"
  prometheus = Prometheus::Client.registry

  HTTP_REQUESTS = prometheus.counter(
    :http_requests_total,
    docstring: "Total HTTP requests",
    labels: [:method, :status, :controller, :action]
  )

  DB_QUERY_DURATION = prometheus.histogram(
    :database_query_duration_seconds,
    docstring: "Database query duration",
    labels: [:operation],
    buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5]
  )
end
```

### Health Check Endpoint

```ruby
# config/routes.rb
get "/up", to: proc { [200, {}, ["OK"]] }
get "/health", to: "health#show"

# app/controllers/health_controller.rb
class HealthController < ActionController::API
  def show
    checks = {
      database: database_connected?,
      redis: redis_connected?,
      migrations: migrations_current?
    }
    status = checks.values.all? ? :ok : :service_unavailable
    render json: checks, status: status
  end

  private

  def database_connected?
    ActiveRecord::Base.connection.execute("SELECT 1")
    true
  rescue StandardError
    false
  end

  def redis_connected?
    Redis.current.ping == "PONG"
  rescue StandardError
    false
  end

  def migrations_current?
    !ActiveRecord::Base.connection.migration_context.needs_migration?
  end
end
```

## Security Configuration

### Rack::Attack (Full Configuration)

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Throttle all requests by IP
  throttle("req/ip", limit: 300, period: 5.minutes) do |req|
    req.ip unless req.path.start_with?("/assets")
  end

  # Throttle login attempts by IP
  throttle("logins/ip", limit: 5, period: 20.seconds) do |req|
    req.ip if req.path == "/login" && req.post?
  end

  # Throttle login attempts by email
  throttle("logins/email", limit: 5, period: 5.minutes) do |req|
    if req.path == "/login" && req.post?
      req.params.dig("user", "email")&.downcase&.strip
    end
  end

  # Throttle API requests by token
  throttle("api/token", limit: 100, period: 1.minute) do |req|
    req.env["HTTP_AUTHORIZATION"]&.split(" ")&.last if req.path.start_with?("/api/")
  end

  # Block suspicious requests
  blocklist("block bad IPs") do |req|
    Rack::Attack::Allow2Ban.filter(req.ip, maxretry: 10, findtime: 1.minute, bantime: 1.hour) do
      req.path == "/login" && req.post?
    end
  end

  # Custom throttle response
  self.throttled_responder = lambda do |matched, _env|
    now = Time.now.utc
    headers = {
      "Content-Type" => "application/json",
      "Retry-After" => (now + matched[:period]).httpdate
    }
    body = { error: "Rate limit exceeded. Retry later." }.to_json
    [429, headers, [body]]
  end
end
```

### SSL / TLS / HSTS

```ruby
# config/environments/production.rb
config.force_ssl = true
config.ssl_options = {
  hsts: {
    subdomains: true,
    preload: true,
    expires: 1.year
  },
  redirect: {
    exclude: ->(request) { request.path.start_with?("/health") }
  }
}
```

### Content Security Policy

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self
    policy.font_src    :self, "https://fonts.gstatic.com"
    policy.img_src     :self, :data, "https:"
    policy.object_src  :none
    policy.script_src  :self
    policy.style_src   :self, "https://fonts.googleapis.com"
    policy.connect_src :self
    policy.frame_ancestors :none
  end
  config.content_security_policy_nonce_generator = ->(request) { request.session.id.to_s }
  config.content_security_policy_nonce_directives = %w[script-src style-src]
end
```

## Database Optimization

### Production Database Config

```yaml
# config/database.yml
production:
  adapter: postgresql
  encoding: unicode
  url: <%= ENV["DATABASE_URL"] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
  timeout: 5000
  reaping_frequency: 10
  connect_timeout: 2
  checkout_timeout: 5
  variables:
    statement_timeout: "30s"
    lock_timeout: "10s"
  prepared_statements: true
```

### CDN Configuration

```ruby
# config/environments/production.rb
config.action_controller.asset_host = ENV["CDN_HOST"]
config.public_file_server.headers = {
  "Cache-Control" => "public, max-age=#{1.year.to_i}",
  "X-Content-Type-Options" => "nosniff"
}
```

### Redis Cache Store

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  expires_in: 1.day,
  namespace: "cache",
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error("Redis error: #{exception}")
    Sentry.capture_exception(exception) if defined?(Sentry)
  }
}
```

## Backup Configuration

```yaml
# config/backup.yml
production:
  database:
    schedule: "0 2 * * *"     # Daily at 2 AM UTC
    retention: 30              # Keep 30 days
    destination: s3://backups/database/
  files:
    schedule: "0 3 * * 0"     # Weekly on Sunday
    retention: 4               # Keep 4 weeks
    paths:
      - public/uploads
      - storage
```

## Deploy Script

```bash
#!/bin/bash
# bin/deploy
set -euo pipefail

echo "==> Running migrations..."
bundle exec rails db:migrate
if [ $? -ne 0 ]; then
  echo "Migration failed! Aborting deploy."
  exit 1
fi

echo "==> Precompiling assets..."
bundle exec rails assets:precompile

echo "==> Restarting application..."
bundle exec pumactl phased-restart

echo "==> Deploy complete!"
```
