# Rails Dev Plugin

> Claude Code plugin with specialized agents and skills for Rails development

[![Version](https://img.shields.io/badge/version-2.0.0-blue.svg)](https://github.com/ag0os/rails-dev-plugin/releases)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

A Claude Code plugin that provides 11 specialized agents and 18 portable skills covering models, controllers, views, services, jobs, testing, architecture, DevOps, GraphQL, Hotwire, API design, and more.

Agents auto-detect your project's stack profile (omakase, service-oriented, or api-first) and coding conventions, then generate code that matches your project — not generic Rails.

## Quick Start

```shell
# In Claude Code
/plugin marketplace add ag0os/rails-dev-plugin
/plugin install rails-dev-plugin@ag0os
```

Restart Claude Code, then just ask:

```
You: "Create a User model with authentication"
You: "Refactor this controller — it has too much business logic"
You: "Add background jobs for email delivery"
```

Claude automatically selects the right agent based on your request.

## What It Does

**Agents** are specialists that handle implementation tasks — one for models, one for controllers, one for testing, etc. They scan your codebase first to match your existing patterns.

**Skills** are portable knowledge that agents draw from. They also work independently in the main conversation (e.g., asking about refactoring patterns or caching strategies).

**Stack profiles** adapt recommendations to how your project is built. A project using Minitest + fixtures + concerns gets different advice than one using RSpec + FactoryBot + service objects.

**Convention detection** goes deeper than profiles — agents detect your specific base classes, naming patterns, result types, auth setup, and more before writing code. They also read your `CLAUDE.md` for intent that overrides detected patterns.

## Agents

| Agent | Domain |
|-------|--------|
| `rails-model` | ActiveRecord models, migrations, associations |
| `rails-controller` | RESTful controllers, routing, params |
| `rails-service` | Service objects, business logic |
| `rails-jobs` | Background jobs, ActiveJob, Sidekiq |
| `rails-views` | ERB templates, partials, ViewComponents |
| `rails-hotwire` | Stimulus controllers, Turbo frames/streams |
| `rails-graphql` | GraphQL schema, resolvers, mutations |
| `rails-api` | REST API, serialization, JWT |
| `rails-test` | RSpec, Minitest, system tests |
| `rails-architect` | Architecture planning, design decisions |
| `rails-devops` | Docker, CI/CD, deployment, monitoring |

## Team Setup

Add to your project's `.claude/settings.json` so the plugin auto-installs for all team members:

```json
{
  "plugins": {
    "marketplaces": [
      {
        "name": "rails-dev",
        "source": "ag0os/rails-dev-plugin"
      }
    ],
    "installed": ["rails-dev-plugin@rails-dev"],
    "autoInstall": true
  }
}
```

## Management

```shell
/plugin disable rails-dev-plugin@ag0os    # Disable
/plugin enable rails-dev-plugin@ag0os     # Re-enable
/plugin uninstall rails-dev-plugin@ag0os  # Uninstall
```

## Contributing

Contributions welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).

---

Built for [Claude Code](https://claude.com/claude-code) and the Rails community.
