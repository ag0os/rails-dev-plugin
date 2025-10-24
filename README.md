# Rails Dev Plugin

> Comprehensive Claude Code plugin with 12 specialized agents for Rails development

[![Version](https://img.shields.io/badge/version-1.1.0-blue.svg)](https://github.com/your-username/rails-dev-plugin/releases)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

A powerful Claude Code plugin that provides 12 specialized AI agents and 2 autonomous Skills to assist with all aspects of Rails application development. From models and controllers to architecture decisions and DevOps, this plugin has you covered.

## 🚀 Quick Start

### Installation

Install directly from GitHub in any Rails project:

```shell
# In Claude Code
/plugin marketplace add your-username/rails-dev-plugin
/plugin install rails-dev-plugin@your-username
```

Replace `your-username` with the GitHub username/organization where you published this plugin.

### First Use

After installation, restart Claude Code and start using the agents:

```
You: "Create a new User model with authentication"
Claude: I'll use the rails-model agent to help...
```

## ✨ Features

### 🤖 12 Specialized Agents

This plugin includes expert agents for every aspect of Rails development.

### 🎯 2 Autonomous Skills

Skills that Claude invokes automatically based on task context:

- **Ruby Refactoring Expert** - Code smell identification, refactoring patterns, Ruby best practices
- **Rails Architecture** - Architectural guidance, design patterns, service layer decisions

### Agent Catalog

#### **Backend Development**
- **rails-model** - ActiveRecord models, migrations, associations, validations, and database schema design
- **rails-controller** - RESTful controllers, authentication/authorization, Pundit policies
- **rails-service** - Service objects, business logic extraction, command/query patterns
- **rails-jobs** - Background jobs, Active Job, Sidekiq, job scheduling

#### **Frontend Development**
- **rails-views** - ViewComponents, ERB templates, TailwindCSS, modern UI patterns
- **rails-stimulus-turbo** - Hotwire, Stimulus controllers, Turbo frames/streams

#### **API Development**
- **rails-graphql** - GraphQL schema design, resolvers, mutations, query optimization

#### **Testing & Quality**
- **rails-test** - TDD with Minitest/RSpec, test coverage, fixtures, system tests

#### **Architecture & Operations**
- **rails-architect** - Architectural decisions, design patterns, system structure
- **rails-devops** - Deployment, CI/CD, Docker, performance, monitoring

#### **Project Management**
- **project-manager-backlog** - Task creation and management using Backlog.md CLI
- **backlog-task-coordinator** - Task analysis and multi-agent coordination

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

### Agent Quick Reference

| Agent | Best For |
|-------|----------|
| `rails-model` | Creating/modifying models, migrations, associations, validations |
| `rails-controller` | Building controllers, handling requests, authorization |
| `rails-views` | Creating views, components, frontend styling |
| `rails-service` | Extracting business logic, service objects |
| `rails-jobs` | Background processing, async tasks |
| `rails-test` | Writing tests, improving coverage |
| `rails-stimulus-turbo` | Adding interactivity, Hotwire features |
| `rails-graphql` | Building GraphQL APIs |
| `rails-architect` | Architecture decisions, design reviews |
| `rails-devops` | Deployment, CI/CD, infrastructure |
| `project-manager-backlog` | Creating and managing tasks |
| `backlog-task-coordinator` | Coordinating multiple agents |

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

## 🛠 Installation Options

### Option 1: From GitHub (Recommended)

```shell
/plugin marketplace add your-username/rails-dev-plugin
/plugin install rails-dev-plugin@your-username
```

### Option 2: From Local Clone

```bash
git clone https://github.com/your-username/rails-dev-plugin.git
cd your-rails-project
claude
```

```shell
/plugin marketplace add /path/to/rails-dev-plugin
/plugin install rails-dev-plugin@local
```

### Option 3: Team-Wide Auto-Install

Add to your project's `.claude/settings.json`:

```json
{
  "plugins": {
    "marketplaces": [
      {
        "name": "rails-dev-tools",
        "source": "your-username/rails-dev-plugin"
      }
    ],
    "installed": [
      "rails-dev-plugin@rails-dev-tools"
    ]
  }
}
```

Team members who trust the repository will automatically get the plugin.

## 🎛 Management

### Enable/Disable

```shell
# Disable (keeps installed)
/plugin disable rails-dev-plugin@your-username

# Re-enable
/plugin enable rails-dev-plugin@your-username
```

### Update

```shell
/plugin uninstall rails-dev-plugin@your-username
/plugin install rails-dev-plugin@your-username
```

### Uninstall

```shell
/plugin uninstall rails-dev-plugin@your-username
```

## 🏗 Plugin Structure

```
rails-dev-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata
├── agents/                   # 12 specialized agents
│   ├── backlog-task-coordinator.md
│   ├── project-manager-backlog.md
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
├── skills/                   # 2 autonomous Skills
│   ├── ruby-refactoring/
│   │   └── SKILL.md
│   └── rails-architecture/
│       └── SKILL.md
├── CHANGELOG.md             # Version history
├── CONTRIBUTING.md          # Contribution guidelines
├── LICENSE                  # MIT License
└── README.md               # This file
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

## 📋 Requirements

- Claude Code installed and running
- Rails project (any version, but optimized for Rails 7+)
- Ruby 3.0+ recommended

## 🐛 Troubleshooting

### Plugin Not Found

```
Error: Plugin not found
```

**Solution**: Verify the marketplace was added correctly:
```shell
/plugin marketplace add your-username/rails-dev-plugin
```

### Agents Not Available

**Solution**: Restart Claude Code after installation.

### Agent Not Responding Correctly

**Solution**: Check the agent's markdown file for proper formatting and update the plugin:
```shell
/plugin uninstall rails-dev-plugin@your-username
/plugin install rails-dev-plugin@your-username
```

## 📚 Documentation

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

- **Issues**: [GitHub Issues](https://github.com/your-username/rails-dev-plugin/issues)
- **Discussions**: [GitHub Discussions](https://github.com/your-username/rails-dev-plugin/discussions)
- **Updates**: Watch this repository for updates

## ⭐️ Show Your Support

If this plugin helps your Rails development workflow, please consider:
- Starring the repository ⭐️
- Sharing it with your team
- Contributing improvements
- Reporting issues or suggestions

---

**Made with ❤️ for the Rails community**
