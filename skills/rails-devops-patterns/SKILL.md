---
name: rails-devops-patterns
description: Analyzes Rails deployment, infrastructure, and production configuration for best practices. Use when setting up Docker, writing CI/CD pipelines, configuring Puma, adding monitoring or logging, hardening security (SSL, Rack::Attack, headers), optimizing database config, or reviewing production environment files. NOT for application business logic, model design, or test writing.
allowed-tools: Read, Grep, Glob
---

# Rails DevOps Patterns

Analyze and recommend best practices for Rails deployment, containerization, CI/CD, monitoring, security, and production optimization.

Follow standard Docker, CI/CD, and Puma conventions. Focus on the opinionated Rails-specific patterns below.

See [patterns.md](patterns.md) for full configuration examples.

## Quick Reference

| Area | Key Files | Tool/Service |
|------|-----------|-------------|
| Docker | `Dockerfile`, `docker-compose.yml` | Docker, BuildKit |
| CI/CD | `.github/workflows/ci.yml` | GitHub Actions |
| Server | `config/puma.rb` | Puma |
| Security | `config/initializers/rack_attack.rb` | Rack::Attack |
| Monitoring | `config/environments/production.rb` | Lograge, structured JSON |
| Deploy | `bin/deploy`, `config/deploy/` | Kamal, Capistrano |

## Core Principles

1. **Immutable deploys**: Build once, deploy the same artifact everywhere
2. **Non-root containers**: Always run as a non-root USER in production
3. **Health checks at /up**: Load balancers and orchestrators need them
4. **Structured JSON logging**: Machine-parseable logs to STDOUT
5. **Defense in depth**: SSL + rate limiting + CSP headers + statement timeouts

## Production Dockerfile (Alpine, Non-Root)

The key non-obvious elements: `SECRET_KEY_BASE=precompile_placeholder` for asset compilation, non-root user, and HEALTHCHECK directive.

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

## Health Check Endpoint

```ruby
# config/routes.rb — minimal for load balancers
get "/up", to: proc { [200, {}, ["OK"]] }
# Detailed check (database, redis, migrations) — see patterns.md
get "/health", to: "health#show"
```

## Structured JSON Logging

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

## SSL with Health Check Exclusion

```ruby
# config/environments/production.rb
config.force_ssl = true
config.ssl_options = {
  hsts: { subdomains: true, preload: true, expires: 1.year },
  redirect: { exclude: ->(request) { request.path.start_with?("/health") } }
}
```

## Production Environment Variables

```bash
RAILS_ENV=production
RAILS_LOG_TO_STDOUT=true
RAILS_SERVE_STATIC_FILES=true
SECRET_KEY_BASE=<generated>
DATABASE_URL=postgres://user:pass@host:5432/dbname
REDIS_URL=redis://host:6379/0
RAILS_MAX_THREADS=5
WEB_CONCURRENCY=2
```

## Database Production Config

Key non-default settings: `statement_timeout` and `lock_timeout` prevent runaway queries.

```yaml
production:
  adapter: postgresql
  url: <%= ENV["DATABASE_URL"] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
  variables:
    statement_timeout: "30s"
    lock_timeout: "10s"
  prepared_statements: true
```

## Zero-Downtime Deployment

- Use `pumactl phased-restart` for rolling restarts (workers restart one at a time)
- Run `db:migrate` before restarting app servers
- Ensure migrations are backward-compatible (add column, then backfill, then add constraint)
- Health check endpoint lets load balancer drain connections

## Anti-Patterns

| Bad | Good | Why |
|-----|------|-----|
| Secrets in ENV files committed to git | `rails credentials:edit` | Security |
| `latest` Docker tag in production | Pinned image versions | Reproducibility |
| No health check endpoint | `GET /up` returning 200 | Load balancer needs it |
| Running as root in container | `adduser` + `USER rails` | Security |
| Logging to files in containers | Log to STDOUT as JSON | Container best practice |
| No statement_timeout | `statement_timeout: '30s'` | Prevent runaway queries |

## Output Format

When reporting on DevOps configuration, use:

```
## DevOps Analysis: [area]

**Current State:**
- summary of what exists

**Issues:**
- [severity] description

**Recommendations:**
1. actionable recommendation with file path
```
