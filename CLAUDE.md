# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Code plugin that provides 10 specialized agents and 1 autonomous Skill for Rails application development. The plugin is distributed via Claude Code's plugin marketplace and installed into Rails projects.

## Repository Structure

```
rails-dev-plugin/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata (name, version, description)
├── agents/                   # 10 specialized agents (markdown files)
│   ├── rails-model.md
│   ├── rails-controller.md
│   ├── rails-views.md
│   ├── rails-service.md
│   ├── rails-jobs.md
│   ├── rails-test.md
│   ├── rails-stimulus-turbo.md
│   ├── rails-graphql.md
│   ├── rails-architect.md
│   └── rails-devops.md
└── skills/                   # 1 autonomous Skill
    └── ruby-refactoring/
        ├── SKILL.md
        ├── code-smells.md
        └── refactoring-patterns.md
```

## Architecture

### Agent System
Each agent is a standalone markdown file with YAML frontmatter that defines:
- `name`: Agent identifier used by Claude Code
- `description`: When Claude should invoke this agent (contains examples)
- `model`: Which Claude model to use (typically "sonnet")
- `color`: UI color for the agent

The description field is critical - it contains trigger patterns and examples that teach Claude Code when to invoke each agent. These examples use a specific format with `<example>`, `<commentary>`, and context markers.

### Skills System
Skills are autonomous capabilities that Claude invokes automatically based on task context. Unlike agents (which are explicitly called), Skills are:
- Invoked proactively by Claude based on the `description` field
- Limited to specific tools via `allowed-tools` directive
- Organized in directories with supporting documentation files

**Ruby Refactoring Expert Skill**: Analyzes code quality, identifies smells, suggests refactoring patterns

## Development Workflow

### Testing Changes Locally

1. Make changes to agent/skill markdown files
2. Install locally in a test Rails project:
   ```bash
   cd /path/to/test-rails-project
   claude
   ```
   ```shell
   /plugin marketplace add /path/to/rails-dev-plugin
   /plugin install rails-dev-plugin@local
   ```
3. Test the agent by triggering it with relevant prompts
4. Restart Claude Code if changes don't appear

### Version Management

- Update `version` in `.claude-plugin/plugin.json`
- Document changes in `CHANGELOG.md` following Keep a Changelog format
- Use semantic versioning (MAJOR.MINOR.PATCH)

### Agent Development Guidelines

**Frontmatter Requirements:**
- Include clear, specific examples in the `description` field
- Examples should show user prompts and expected agent invocation
- Use `<commentary>` blocks to explain why the agent should be used

**Content Structure:**
- Start with "You are a [role] expert..." to establish agent identity
- Include "Core Responsibilities" section
- Provide concrete best practices and guidelines
- Add code examples where helpful
- Reference relevant directories (app/models, app/controllers, etc.)

**Key Principles:**
- Keep agents focused on specific domains
- Provide actionable, step-by-step guidance
- Include Rails/Ruby best practices
- Consider project conventions (Jumpstart Pro, ViewComponent, etc.)

### Skills Development Guidelines

**Frontmatter for Skills:**
- `name`: Descriptive skill name
- `description`: When Claude should invoke (must be comprehensive)
- `allowed-tools`: Restrict to Read, Grep, Glob for analysis tasks

**Directory Structure:**
- Main `SKILL.md` contains core skill logic
- Supporting `.md` files provide reference material
- Skills can reference other files in their directory

## Plugin Distribution

This plugin is designed to be installed via:
1. GitHub marketplace: `/plugin marketplace add user/rails-dev-plugin`
2. Local development: `/plugin marketplace add /path/to/rails-dev-plugin`
3. Team auto-install: Add to project's `.claude/settings.json`

## Key Conventions

- All agent files use `.md` extension with YAML frontmatter
- Plugin metadata lives in `.claude-plugin/plugin.json`
- Version numbers follow semantic versioning
- CHANGELOG.md tracks all changes
- Agent colors: Use for visual distinction in Claude Code UI
- Model selection: "sonnet" for most agents (balance of speed/capability)

## Testing Considerations

When modifying agents:
- Test in a real Rails project, not just this repository
- Verify the `description` triggers correctly
- Check for conflicts with other agents
- Test both explicit requests ("use rails-model agent") and implicit triggers
- Ensure examples in agent content actually work

## Important Notes

- This is a **plugin repository**, not a Rails application
- No Ruby/Rails code runs here - agents provide guidance for other projects
- The markdown files define agent behavior and knowledge
- Changes to `.claude-plugin/plugin.json` require plugin reinstallation
- Changes to agent markdown files may require Claude Code restart
