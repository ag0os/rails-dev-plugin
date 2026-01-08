# Changelog

All notable changes to the Rails Dev Plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned
- Video tutorials for agent usage
- Integration with popular Rails gems
- Example test project with sample code

## [1.3.0] - 2026-01-06

### Added

#### New Skill: Ruby Object Design Expert
- **`ruby-object-design` Skill** - Autonomous guidance for Ruby object-oriented design decisions
  - Main `SKILL.md` with comprehensive decision framework
  - `class-vs-module.md` - When to use class vs module with concrete examples
  - `data-structures.md` - Patterns for Struct, Data (Ruby 3.2+), OpenStruct, and Hash
- Core principle from Dave Thomas's "Rubyist's Mandate": Ruby is object-oriented, not class-oriented
- Decision tree for choosing the right construct (class, module, Struct, Data, Hash)
- Automatic Ruby version detection for Data class recommendations

#### Context-Aware Pattern System
All enhanced skills now feature automatic project detection:
- **Gem Detection** - Automatically detects Pundit, CanCanCan, Devise, Sidekiq, Solid Queue, etc.
- **Ruby Version Detection** - Adapts patterns for Ruby 3.2+ (Data class) vs earlier versions
- **Rails Version Detection** - Uses `params.expect` (7.1+) vs `params.require.permit`
- **Turbo Version Detection** - Applies morphing patterns (8.0+) with fallbacks
- **Multi-tenant Detection** - Enables CurrentAttributes patterns when detected
- **SPA Detection** - Adjusts Hotwire patterns when React/Vue/Angular detected

#### Enhanced Rails Controller Patterns
- **Resource-based Controllers** - 37signals-style single-resource controllers pattern
- **Layered Controller Concerns** - Composable authentication, authorization, and pagination
- **Model-based Authorization** - Context-aware (respects existing Pundit/CanCanCan)
- **`params.expect` Pattern** - Rails 7.1+ with automatic fallback for earlier versions
- Context awareness reference section documenting all detection points

#### Enhanced Rails Jobs Patterns
- **`_later`/`_now` Naming Convention** - Clear async vs sync method naming
- **Shallow Jobs Pattern** - Jobs that delegate to model methods (37signals style)
- **Automatic Context Serialization** - Multi-tenant CurrentAttributes preservation
- **Concurrency Control** - Solid Queue and Sidekiq-specific patterns
- **Recurring Jobs Configuration** - Cron-style job scheduling patterns
- **Batch Job Enqueueing** - Efficient bulk job creation patterns

#### Enhanced Hotwire Patterns
- **Turbo Morphing** - Turbo 8.0+ page refresh with morph method
- **Permanent Elements** - Preserving state during morphs (`data-turbo-permanent`)
- **Fragment Caching with Turbo** - Cache-aware Turbo Stream responses
- **Auto-save Controller** - Stimulus controller with debounced form submission
- **JavaScript Helpers** - `debounce`, `nextFrame`, `waitForElement` utilities
- Context awareness for Hotwire vs SPA detection

#### Enhanced Rails Model Patterns
- **Default Association Values** - Context-aware for CurrentAttributes/multi-tenant
- **Association Extensions** - Custom methods on associations
- **Self-referential Convenience Methods** - Cleaner model APIs
- **New `value-objects.md`** - Comprehensive Struct/Data patterns with Ruby version detection

### Changed
- All enhanced skills now include "Context Awareness" reference sections
- Patterns provide automatic fallbacks for version-dependent features
- Conflict detection warns when multiple solutions exist (e.g., Pundit + CanCanCan)
- Skills detect project conventions before suggesting patterns

### Technical Details
- Integrated patterns from Dave Thomas's "The Rubyist's Mandate"
- Incorporated 37signals Fizzy Architecture production patterns
- All version-dependent patterns include detection instructions
- Fallback patterns provided for older Rails/Ruby/Turbo versions

### Migration Notes
For users upgrading from 1.2.0:
- No breaking changes - all existing functionality preserved
- New `ruby-object-design` Skill activates automatically on relevant queries
- Enhanced skills provide smarter, context-aware recommendations
- Detection is automatic - no configuration required

## [1.2.0] - 2025-10-27

### Added
- **3 New Autonomous Skills**:
  - `Rails Performance Analyzer` - N+1 query detection, bottleneck identification, optimization recommendations
  - `Rails Security Auditor` - Security vulnerability scanning, SQL injection detection, authorization review
  - `Rails Upgrade Assistant` - Rails version upgrade planning, deprecation handling, breaking change guidance
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

### Migration Notes
For users upgrading from 1.1.0:
- No breaking changes
- Existing agents work as before but with enhanced triggering
- New Skills activate automatically
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

[Unreleased]: https://github.com/ag0os/rails-dev-plugin/compare/v1.3.0...HEAD
[1.3.0]: https://github.com/ag0os/rails-dev-plugin/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/ag0os/rails-dev-plugin/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/ag0os/rails-dev-plugin/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/ag0os/rails-dev-plugin/releases/tag/v1.0.0
