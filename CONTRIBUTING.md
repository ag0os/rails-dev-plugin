# Contributing to Rails Dev Plugin

Thank you for your interest in contributing to the Rails Dev Plugin! This document provides guidelines and instructions for contributing.

## 🌟 Ways to Contribute

- **Improve existing agents** - Enhance agent prompts and capabilities
- **Add new agents** - Create specialized agents for specific Rails tasks
- **Fix bugs** - Report or fix issues you encounter
- **Documentation** - Improve README, guides, or add examples
- **Share feedback** - Let us know how the plugin works for you

## 🚀 Getting Started

### Prerequisites

- Git installed
- Claude Code installed
- A Rails project for testing
- Basic understanding of Claude Code plugins

### Setup for Development

1. **Fork the repository**
   ```bash
   # Click "Fork" on GitHub, then:
   git clone https://github.com/your-username/rails-dev-plugin.git
   cd rails-dev-plugin
   ```

2. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **Install locally for testing**
   ```bash
   # In a Rails project
   claude
   ```
   ```shell
   /plugin marketplace add /path/to/rails-dev-plugin
   /plugin install rails-dev-plugin@local
   ```

## 📝 Agent Development Guidelines

### Agent File Structure

Each agent is a markdown file in the `agents/` directory with frontmatter:

```markdown
---
name: agent-name
description: Clear description of when to use this agent
model: sonnet
color: blue
---

You are a specialized [role] expert...

## Your Core Responsibilities

1. First responsibility
2. Second responsibility
...
```

### Writing Effective Agents

1. **Clear Description**: The `description` field should clearly state when Claude Code should invoke this agent
2. **Focused Expertise**: Keep each agent focused on a specific domain
3. **Actionable Guidance**: Provide concrete, step-by-step guidance
4. **Best Practices**: Include Rails and Ruby best practices
5. **Examples**: Add code examples where helpful
6. **Context Awareness**: Reference the project structure and conventions

### Agent Naming Conventions

- Use lowercase with hyphens: `rails-model`, `rails-controller`
- Be descriptive but concise
- Prefix with domain when applicable: `rails-*`

## 🧪 Testing Your Changes

Before submitting a pull request:

1. **Test the agent** in a real Rails project
2. **Verify the description** triggers the agent correctly
3. **Check for conflicts** with other agents
4. **Update documentation** if needed
5. **Test edge cases** and error scenarios

### Testing Checklist

- [ ] Agent activates when expected
- [ ] Agent provides helpful, accurate guidance
- [ ] Examples in agent work correctly
- [ ] No typos or formatting issues
- [ ] Agent doesn't conflict with others
- [ ] Documentation updated if needed

## 📤 Submitting Changes

### Pull Request Process

1. **Update the CHANGELOG.md**
   ```markdown
   ## [Unreleased]
   ### Added
   - New rails-awesome agent for awesome functionality
   ```

2. **Commit your changes**
   ```bash
   git add .
   git commit -m "Add rails-awesome agent for awesome functionality"
   ```

3. **Push to your fork**
   ```bash
   git push origin feature/your-feature-name
   ```

4. **Open a Pull Request**
   - Go to the original repository on GitHub
   - Click "New Pull Request"
   - Select your fork and branch
   - Fill out the PR template

### Pull Request Guidelines

- **Title**: Clear, concise description of the change
- **Description**: Explain what changed and why
- **Testing**: Describe how you tested the changes
- **Screenshots**: Include if relevant (especially for documentation changes)
- **Breaking Changes**: Clearly mark any breaking changes

### Commit Message Format

Use clear, descriptive commit messages:

```
Add rails-performance agent for optimization tasks

- Covers database query optimization
- Includes caching strategies
- Provides profiling guidance
```

## 🐛 Reporting Bugs

### Before Submitting a Bug Report

- Check existing issues to avoid duplicates
- Test with the latest version
- Gather reproduction steps

### Bug Report Template

```markdown
**Description**
A clear description of the bug

**To Reproduce**
1. Install plugin
2. Run command '...'
3. See error

**Expected Behavior**
What should have happened

**Actual Behavior**
What actually happened

**Environment**
- Claude Code version:
- Rails version:
- Ruby version:
- Plugin version:

**Additional Context**
Any other relevant information
```

## 💡 Feature Requests

We welcome feature requests! Please:

1. **Check existing requests** to avoid duplicates
2. **Describe the use case** - why is this needed?
3. **Provide examples** - how would it work?
4. **Consider alternatives** - are there other ways to solve this?

## 📋 Code Style

### Agent Markdown Style

- Use proper markdown formatting
- Keep lines under 120 characters when possible
- Use consistent indentation (2 spaces)
- Add blank lines between sections
- Use code fences with language tags

### Example Style

```markdown
## Core Responsibilities

1. **First item** - Description with proper emphasis
2. **Second item** - Another clear description

### Subsection

Provide clear examples:

```ruby
# Good example
class User < ApplicationRecord
  validates :email, presence: true
end
```

## 🤝 Code of Conduct

### Our Standards

- Be respectful and inclusive
- Provide constructive feedback
- Focus on what's best for the community
- Show empathy towards others

### Unacceptable Behavior

- Harassment or discrimination
- Trolling or inflammatory comments
- Publishing private information
- Unprofessional conduct

## 📞 Getting Help

- **Questions**: Open a GitHub Discussion
- **Issues**: Create a GitHub Issue
- **Ideas**: Start a Discussion or open an Issue

## 🎉 Recognition

Contributors will be:
- Listed in release notes
- Acknowledged in the project
- Appreciated by the community!

## 📄 License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

Thank you for contributing to Rails Dev Plugin! 🙏
