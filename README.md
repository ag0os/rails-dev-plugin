# Rails Dev Plugin

> Comprehensive Claude Code plugin with specialized agents for Rails development

[![Version](https://img.shields.io/badge/version-1.2.0-blue.svg)](https://github.com/ag0os/rails-dev-plugin/releases)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

A Claude Code plugin that provides specialized AI agents and autonomous Skills to assist with all aspects of Rails application development. From models and controllers to architecture decisions and DevOps, this plugin has you covered.

## 🚀 Quick Start

### Installation

Install directly from GitHub in any Rails project:

```shell
# In Claude Code
/plugin marketplace add ag0os/rails-dev-plugin
/plugin install rails-dev-plugin@ag0os
```

### Team-Wide Auto-Install

For teams, add to your Rails project's `.claude/settings.json`:

```bash
# Copy the example template
mkdir -p .claude
cp examples/project-settings/rails-project.json .claude/settings.json
git add .claude/settings.json
git commit -m "Enable Rails Dev Plugin for team"
```

Or create manually:

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

Commit this file to your repository. Team members will automatically get the plugin on first Claude Code launch.

### Per-Project Control

To disable the plugin in non-Rails projects, see [examples/project-settings/](examples/project-settings/) for template configurations.

### First Use

After installation, restart Claude Code and start using the agents:

```
You: "Create a new User model with authentication"
Claude: I'll use the rails-model agent to help...
```

## ✨ Features

### 🤖 10 Specialized Agents

Expert agents for every aspect of Rails development:

| Agent | Domain | Specialization |
|-------|--------|----------------|
| `rails-model` | Backend | ActiveRecord models, migrations, associations, validations, schema design |
| `rails-controller` | Backend | RESTful controllers, authentication/authorization, Pundit policies |
| `rails-service` | Backend | Service objects, business logic extraction, command/query patterns |
| `rails-jobs` | Backend | Background jobs, Active Job, Sidekiq, job scheduling |
| `rails-views` | Frontend | ViewComponents, ERB templates, TailwindCSS, modern UI patterns |
| `rails-stimulus-turbo` | Frontend | Hotwire, Stimulus controllers, Turbo frames/streams |
| `rails-graphql` | API | GraphQL schema design, resolvers, mutations, query optimization |
| `rails-test` | Testing | TDD with Minitest/RSpec, test coverage, fixtures, system tests |
| `rails-architect` | Architecture | Architectural decisions, design patterns, system structure |
| `rails-devops` | Operations | Deployment, CI/CD, Docker, performance, monitoring |

### 🎯 1 Autonomous Skill

- **Ruby Refactoring Expert** - Code smell identification, refactoring patterns, Ruby best practices

## 📖 Usage

### Automatic Agent Selection

Claude Code automatically selects the appropriate agent based on your request:

```
You: "Add validations to the Order model"
→ Uses rails-model agent

You: "Create a controller for managing products"
→ Uses rails-controller agent

You: "Design a service object for payment processing"
→ Uses rails-architect + rails-service agents
```

### Explicit Agent Requests

You can also explicitly request specific agents:

```
You: "Use the rails-architect agent to review my service layer design"
You: "Have the rails-test agent help me improve test coverage"
```

## 🎯 Use Cases

### Building New Features

```
You: "I need to add a blog system with posts, comments, and tags"
→ Agents coordinate to create models, controllers, views, and tests
```

### Refactoring

```
You: "This controller has too much business logic. Help me refactor it."
→ rails-architect analyzes, rails-service extracts logic, rails-test ensures coverage
```

### Performance Optimization

```
You: "The posts#index action is slow with 1000+ records"
→ rails-architect + rails-devops suggest pagination, caching, and optimization
```

### Test Coverage

```
You: "I need tests for the PaymentProcessor service"
→ rails-test creates comprehensive test suite with edge cases
```

## 🎛 Management

### Enable/Disable

```shell
# Disable (keeps installed)
/plugin disable rails-dev-plugin@ag0os

# Re-enable
/plugin enable rails-dev-plugin@ag0os
```

### Update

```shell
/plugin uninstall rails-dev-plugin@ag0os
/plugin install rails-dev-plugin@ag0os
```

### Uninstall

```shell
/plugin uninstall rails-dev-plugin@ag0os
```

## 🏗 Plugin Structure

```
rails-dev-plugin/
├── .claude-plugin/
│   ├── plugin.json                # Plugin metadata
│   └── marketplace.json           # Marketplace configuration
├── .mcp.json                      # MCP server configuration
├── agents/                         # 10 specialized agents
│   ├── rails-architect.md
│   ├── rails-controller.md
│   ├── rails-devops.md
│   ├── rails-graphql.md
│   ├── rails-jobs.md
│   ├── rails-model.md
│   ├── rails-service.md
│   ├── rails-stimulus-turbo.md
│   ├── rails-test.md
│   └── rails-views.md
├── skills/                         # 1 autonomous Skill
│   └── ruby-refactoring/
│       ├── SKILL.md
│       ├── code-smells.md
│       └── refactoring-patterns.md
├── docs/
│   ├── quick-start.md             # Common workflows
│   └── agent-decision-tree.md     # Agent selection guide
├── examples/
│   └── project-settings/          # Example .claude/settings.json files
│       ├── rails-project.json
│       ├── non-rails-project.json
│       └── README.md
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

## 🤝 Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for details.

### Quick Contribution Guide

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-agent`)
3. Make your changes to agent markdown files
4. Test the agent in a Rails project
5. Commit your changes (`git commit -m 'Add amazing new agent'`)
6. Push to the branch (`git push origin feature/amazing-agent`)
7. Open a Pull Request

## 🐛 Troubleshooting

### Plugin Not Found

```
Error: Plugin not found
```

**Solution**: Verify the marketplace was added correctly:
```shell
/plugin marketplace add ag0os/rails-dev-plugin
```

### Agents Not Available

**Solution**: Restart Claude Code after installation.

### Agent Not Responding Correctly

**Solution**: Check the agent's markdown file for proper formatting and update the plugin:
```shell
/plugin uninstall rails-dev-plugin@ag0os
/plugin install rails-dev-plugin@ag0os
```

## 📚 Documentation

- **[Quick Start Guide](docs/quick-start.md)** - Common workflows and examples
- **[Agent Decision Tree](docs/agent-decision-tree.md)** - Which agent to use when
- [Agent Development Guide](docs/agent-development.md) *(coming soon)*
- [Best Practices](docs/best-practices.md) *(coming soon)*
- [FAQ](docs/faq.md) *(coming soon)*

## 🔄 Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.

## 📄 License

MIT License - see [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- Built for [Claude Code](https://claude.com/claude-code)
- Inspired by Rails best practices and community conventions
- Designed for modern Rails development workflows

## 💬 Support

- **Issues**: [GitHub Issues](https://github.com/ag0os/rails-dev-plugin/issues)
- **Discussions**: [GitHub Discussions](https://github.com/ag0os/rails-dev-plugin/discussions)
- **Updates**: Watch this repository for updates
- **Quick Start**: [docs/quick-start.md](docs/quick-start.md)
- **Agent Guide**: [docs/agent-decision-tree.md](docs/agent-decision-tree.md)

## ⭐️ Show Your Support

If this plugin helps your Rails development workflow, please consider:
- Starring the repository ⭐️
- Sharing it with your team
- Contributing improvements
- Reporting issues or suggestions

---

**Made with ❤️ for the Rails community**
