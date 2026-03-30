# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Code plugin that provides 11 specialized agents and 17 portable skills for Rails application development. Agents are lean orchestrators that load domain-specific skills via the `skills` frontmatter field. Skills are independently usable and exportable to other platforms.

## Repository Structure

```
rails-dev-plugin/
├── .claude-plugin/
│   └── plugin.json                     # Plugin metadata (name, version, description)
├── agents/                              # 11 lean agents (reference skills for knowledge)
│   ├── rails-model.md                   # skills: [rails-model-patterns]
│   ├── rails-controller.md              # skills: [rails-controller-patterns]
│   ├── rails-views.md                   # skills: [rails-views-patterns]
│   ├── rails-service.md                 # skills: [rails-service-patterns]
│   ├── rails-jobs.md                    # skills: [rails-jobs-patterns]
│   ├── rails-test.md                    # skills: [rails-testing-patterns]
│   ├── rails-hotwire.md                 # skills: [hotwire-patterns]
│   ├── rails-graphql.md                 # skills: [rails-graphql-patterns]
│   ├── rails-architect.md               # skills: [rails-architecture-patterns]
│   ├── rails-devops.md                  # skills: [rails-devops-patterns]
│   └── rails-api.md                     # skills: [rails-api-patterns, rails-controller-patterns]
└── skills/                              # 17 portable skills (domain knowledge)
    ├── rails-model-patterns/            # ActiveRecord, associations, migrations
    ├── rails-controller-patterns/       # RESTful controllers, routing, params
    ├── rails-views-patterns/            # ERB, partials, helpers, caching
    ├── rails-service-patterns/          # Service objects, result pattern, DI
    ├── rails-jobs-patterns/             # ActiveJob, Sidekiq, idempotency
    ├── rails-testing-patterns/          # RSpec, Minitest, system tests
    ├── hotwire-patterns/                # Stimulus, Turbo frames/streams
    ├── rails-graphql-patterns/          # Schema, mutations, DataLoader
    ├── rails-architecture-patterns/     # Planning, design decisions, skill directory
    ├── rails-devops-patterns/           # Docker, CI/CD, monitoring, security
    ├── rails-api-patterns/              # API controllers, serialization, JWT
    ├── rails-stack-profiles/            # Stack profile detection (omakase, service-oriented, api-first)
    ├── rails-mailer-patterns/           # Action Mailer, email delivery, previews
    ├── rails-auth-patterns/             # Authentication (built-in vs Devise)
    ├── rails-caching-patterns/          # Fragment, low-level, HTTP caching
    ├── ruby-refactoring/                # Code smells, refactoring patterns
    └── ruby-object-design/              # Class vs module, Struct, Data
```

## Architecture

### Key Design Principle

**Skills = portable domain knowledge. Agents = lean execution orchestrators.**

Agents use the `skills` frontmatter field to preload skill content at startup. This means:
- All domain knowledge (patterns, code examples, best practices) lives in skills
- Agent bodies contain only: role identity, execution workflow, completion checklist
- Skills work independently (invoked in main conversation) AND as agent knowledge
- Skills are exportable to other platforms without modification

### Stack Profiles

The plugin supports three Rails stack profiles: **omakase**, **service-oriented**, and **api-first** (plus hybrids). The `rails-stack-profiles` skill defines detection logic and per-profile recommendations. The architect agent detects the profile as Step 0 before making any recommendations.

Other skills and agents should present profile-appropriate patterns when the decision differs by profile (e.g., "extract to concern" for omakase vs "extract to service object" for service-oriented). See `skills/rails-stack-profiles/` for details.

### Agent System

Each agent is a lean markdown file with YAML frontmatter:
- `name`: Agent identifier
- `description`: Trigger patterns with `<example>` blocks
- `model`: "sonnet" for all agents
- `color`: UI distinction
- `tools`: Tool access (Read, Write, Edit, Grep, Glob, Bash, etc.)
- `skills`: List of skill names to preload as domain knowledge

**Agent bodies do NOT contain code examples** — those are in skills.

### Skills System

Each skill is a directory with SKILL.md and supporting docs:
- `name`: kebab-case identifier (must match what agents reference)
- `description`: WHAT + WHEN triggers + NOT FOR negative triggers (≤1024 chars)
- `allowed-tools`: Read, Grep, Glob (for independent invocation)
- SKILL.md ≤ 200 lines; detailed content in supporting `.md` files

Six skills are independent — not tied to any single agent:
- `ruby-refactoring` and `ruby-object-design` (standalone code quality skills)
- `rails-stack-profiles` (loaded by the architect agent, but also usable independently for profile detection)
- `rails-mailer-patterns`, `rails-auth-patterns`, `rails-caching-patterns` (standalone domain skills)

## Development Workflow

### Testing Changes Locally

1. Make changes to agent/skill markdown files
2. Install locally in a test Rails project:
   ```bash
   cd /path/to/test-rails-project
   claude
   ```
   ```
   /plugin marketplace add /path/to/rails-dev-plugin
   /plugin install rails-dev-plugin@local
   ```
3. Test agent delegation triggers correctly
4. Test skills work independently in main conversation
5. Restart Claude Code if changes don't appear

### Version Management

- Update `version` in `.claude-plugin/plugin.json`
- Document changes in `CHANGELOG.md` following Keep a Changelog format
- Use semantic versioning (MAJOR.MINOR.PATCH)

### Agent Development Guidelines

- Keep agents lean — execution workflow and checklist only
- NO code examples in agent bodies (put them in skills)
- Include `<example>` blocks in `description` for trigger patterns
- Reference skills via the `skills` frontmatter field
- Start body with "You are a [role] specialist..."

### Skill Development Guidelines

- `name` MUST be kebab-case (no spaces, no capitals)
- `description` must include WHAT + WHEN + negative triggers
- SKILL.md ≤ 200 lines; use supporting docs for detail
- Include: quick reference table, core principles, code examples, anti-patterns, output format
- Supporting docs go in the same directory (not in subdirectories)
- **Profile-awareness:** When a skill covers a decision that differs by profile (e.g., where business logic goes, which testing framework to use), present the profile-appropriate pattern. Use the format: "**Omakase:** do X / **Service-oriented:** do Y" rather than presenting one approach as universal. Reference `rails-stack-profiles` for detection logic.

## Key Conventions

- All files use `.md` extension with YAML frontmatter
- Plugin metadata in `.claude-plugin/plugin.json`
- Semantic versioning
- Agent colors: distinct per agent for UI clarity
- Model: "sonnet" for all agents

## Important Notes

- This is a **plugin repository**, not a Rails application
- No Ruby/Rails code runs here — agents and skills provide guidance for other projects
- The `skills` field in agent frontmatter causes skill content to be injected into the agent's context at startup
- Plugin agents do NOT support `hooks`, `mcpServers`, or `permissionMode` fields
- Changes to `.claude-plugin/plugin.json` require plugin reinstallation
