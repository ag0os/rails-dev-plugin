---
name: rails-devops-patterns
description: Analyzes Rails deployment, infrastructure, and production configuration for best practices. Use when setting up Docker, writing CI/CD pipelines, configuring Puma, adding monitoring or logging, hardening security (SSL, Rack::Attack, headers), optimizing database config, or reviewing production environment files. NOT for application business logic, model design, or test writing.
allowed-tools: Read, Grep, Glob
---

# Rails DevOps Patterns

Analyze and recommend best practices for Rails deployment, containerization, CI/CD, monitoring, security, and production optimization.

See [patterns.md](patterns.md) for full configuration examples.

## Quick Reference

| Area | Key Files | Tool/Service |
|------|-----------|-------------|
| Docker | `Dockerfile`, `docker-compose.yml` | Docker, BuildKit |
| CI/CD | `.github/workflows/ci.yml` | GitHub Actions |
| Server | `config/puma.rb` | Puma |
| Database | `config/database.yml` | PostgreSQL |
| Cache | `config/environments/production.rb` | Redis |
| Security | `config/initializers/rack_attack.rb` | Rack::Attack |
| Monitoring | `config/initializers/monitoring.rb` | Prometheus, Lograge |
| Secrets | `config/credentials.yml.enc` | Rails credentials |
| Deploy | `bin/deploy`, `config/deploy/` | Kamal, Capistrano |

## Core Principles

1. **Infrastructure as Code** -- every config in version control
2. **Immutable deploys** -- build once, deploy the same artifact everywhere
3. **Twelve-Factor App** -- env vars for config, stateless processes, disposable containers
4. **Defense in depth** -- SSL + rate limiting + strong params + CSP headers
5. **Observe everything** -- structured logging, metrics, health checks
6. **Fail fast, recover faster** -- health endpoints, zero-downtime deploys, automated rollback

## Key Patterns

### Dockerfile (Multi-stage, Alpine)

```dockerfile
FROM ruby:3.2-alpine AS base
RUN apk add --no-cache postgresql-dev tzdata
WORKDIR /app

FROM base AS build
RUN apk add --no-cache build-base git nodejs yarn
COPY Gemfile* ./
RUN bundle config set deployment true && \
    bundle config set without "development test" && \
    bundle install
COPY package.json yarn.lock ./
RUN yarn install --production
COPY . .
RUN bundle exec rails assets:precompile

FROM base AS runtime
COPY --from=build /app /app
EXPOSE 3000
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

### Puma Configuration

```ruby
# config/puma.rb
max_threads = ENV.fetch("RAILS_MAX_THREADS", 5)
threads max_threads, max_threads
port ENV.fetch("PORT", 3000)
environment ENV.fetch("RAILS_ENV", "development")
workers ENV.fetch("WEB_CONCURRENCY", 2)
preload_app!
```

### Rack::Attack Rate Limiting

```ruby
# config/initializers/rack_attack.rb
Rack::Attack.throttle("req/ip", limit: 300, period: 5.minutes) { |req| req.ip }
Rack::Attack.throttle("logins/ip", limit: 5, period: 20.seconds) do |req|
  req.ip if req.path == "/login" && req.post?
end
```

### SSL and Security Headers

```ruby
# config/environments/production.rb
config.force_ssl = true
config.ssl_options = { hsts: { subdomains: true, preload: true, expires: 1.year } }
```

### Structured Logging

```ruby
# config/environments/production.rb
config.logger = ActiveSupport::TaggedLogging.new(
  Logger.new(STDOUT).tap { |l|
    l.formatter = proc { |sev, time, prog, msg|
      { severity: sev, time: time.iso8601, msg: msg, pid: Process.pid }.to_json + "\n"
    }
  }
)
```

## Anti-Patterns

| Bad | Good | Why |
|-----|------|-----|
| Secrets in ENV files committed to git | `rails credentials:edit` | Security |
| `latest` Docker tag in production | Pinned image versions | Reproducibility |
| No health check endpoint | `GET /up` returning 200 | Load balancer needs it |
| Single Puma worker | `WEB_CONCURRENCY >= 2` | Utilize multi-core |
| No rate limiting | Rack::Attack on login + API | Prevent abuse |
| Logging to files in containers | Log to STDOUT | Container best practice |
| No database statement timeout | `statement_timeout: '30s'` | Prevent runaway queries |

## Environment Variables Checklist

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
