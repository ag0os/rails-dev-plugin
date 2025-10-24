# Changelog

All notable changes to the Rails Dev Plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned
- Additional documentation and examples
- Video tutorials for agent usage
- Integration with popular Rails gems

## [1.1.0] - 2025-10-24

### Added
- **Ruby Refactoring Expert Skill** - Autonomous code smell identification and refactoring guidance
- **Rails Architecture Skill** - Automatic architectural decision support and design patterns

### Changed
- Plugin now includes 2 autonomous Skills that Claude invokes based on context
- Updated description to highlight Skills capability

## [1.0.0] - 2025-10-24

### Added
- Initial release of Rails Dev Plugin
- 12 specialized agents for Rails development:
  - `rails-model` - Model development and database design
  - `rails-controller` - Controller and routing expertise
  - `rails-views` - ViewComponent and frontend development
  - `rails-service` - Service object patterns
  - `rails-jobs` - Background job processing
  - `rails-test` - Testing strategies and implementation
  - `rails-stimulus-turbo` - Hotwire and modern JavaScript
  - `rails-graphql` - GraphQL API development
  - `rails-architect` - Architecture and design decisions
  - `rails-devops` - Deployment and operations
  - `project-manager-backlog` - Task management with Backlog.md
  - `backlog-task-coordinator` - Multi-agent task coordination
- Professional README with installation instructions
- MIT License
- Contributing guidelines
- Plugin metadata and structure

### Notes
- Optimized for Rails 7+ applications
- Compatible with Ruby 3.0+
- Supports Jumpstart Pro conventions
- Follows Evil Martians ViewComponent best practices

---

## Version History Legend

- **Added** - New features
- **Changed** - Changes in existing functionality
- **Deprecated** - Soon-to-be removed features
- **Removed** - Removed features
- **Fixed** - Bug fixes
- **Security** - Security improvements

[Unreleased]: https://github.com/your-username/rails-dev-plugin/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/your-username/rails-dev-plugin/releases/tag/v1.0.0
