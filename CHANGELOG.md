# Changelog

All notable changes to the Rails Dev Plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned
- Video tutorials for agent usage
- Integration with popular Rails gems
- Example test project with sample code

## [1.2.0] - 2025-10-27

### Added
- **3 New Autonomous Skills**:
  - `Rails Performance Analyzer` - N+1 query detection, bottleneck identification, optimization recommendations
  - `Rails Security Auditor` - Security vulnerability scanning, SQL injection detection, authorization review
  - `Rails Upgrade Assistant` - Rails version upgrade planning, deprecation handling, breaking change guidance
- **Hooks Configuration** (`hooks/hooks.json`):
  - RuboCop auto-correction on file edits
  - Migration safety reminders
  - Production deployment checklist prompts
  - File deletion warnings
- **MCP Integration** (`.mcp.json`):
  - Rails documentation access (guides.rubyonrails.org, api.rubyonrails.org)
  - Hotwire documentation access (Turbo, Stimulus)
- **Comprehensive Documentation**:
  - Quick Start Guide with common workflows and examples
  - Agent Decision Tree for choosing the right agent
  - Updated README with team setup instructions
  - Compatibility matrix for Rails and Ruby versions
- **GitHub Templates**:
  - Bug report template
  - Feature request template
  - Agent improvement template
  - Pull request template

### Changed
- **All 10 Agents Enhanced**:
  - Added "PROACTIVELY" and "MUST BE USED" language to descriptions for better auto-invocation
  - Added specific trigger keywords for improved agent selection
  - Implemented tools restrictions for security and focus
  - Applied semantic color scheme (blue for backend, green for frontend, red for DevOps, etc.)
  - Added proactive examples showing when agents should auto-trigger
- **Skill Descriptions Enhanced**:
  - Added comprehensive trigger phrases for automatic invocation
  - Improved descriptions for better context detection
- **Plugin Metadata**:
  - Updated version to 1.2.0
  - Enhanced keywords for better discoverability
  - Added repository and license information
  - Updated homepage URLs to ag0os/rails-dev-plugin

### Improved
- Agent invocation now more automatic based on user's natural language
- Skills trigger more reliably on relevant keywords
- Better documentation for team collaboration
- Clearer agent selection guidance

### Technical Details
- Agent colors now follow semantic scheme for better visual organization
- Tools restrictions improve security and agent focus
- MCP servers enable real-time Rails documentation access
- Hooks automate common Rails development workflows

### Migration Notes
For users upgrading from 1.1.0:
- No breaking changes
- Existing agents work as before but with enhanced triggering
- New Skills activate automatically
- Hooks are opt-in (configure in project's `.claude/settings.json`)
- Update installation command to use `ag0os/rails-dev-plugin`

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
- 10 specialized agents for Rails development:
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

[Unreleased]: https://github.com/ag0os/rails-dev-plugin/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/ag0os/rails-dev-plugin/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/ag0os/rails-dev-plugin/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/ag0os/rails-dev-plugin/releases/tag/v1.0.0
