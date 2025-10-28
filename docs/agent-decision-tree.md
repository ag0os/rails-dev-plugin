# Agent Decision Tree

Not sure which agent or Skill to use? Use this decision tree to find the right one for your task.

## Quick Decision Flowchart

```
What are you working on?
│
├─ 📊 Data & Database
│  ├─ Models, migrations, associations? → rails-model
│  ├─ Database performance, N+1 queries? → Rails Performance Analyzer (Skill)
│  └─ Database schema design decisions? → rails-architect
│
├─ 🌐 HTTP & Controllers
│  ├─ Controller actions, routes? → rails-controller
│  ├─ Authorization, Pundit policies? → rails-controller
│  └─ API endpoints, JSON responses? → rails-controller (or rails-graphql for GraphQL)
│
├─ 🎨 Views & Frontend
│  ├─ ERB templates, ViewComponents? → rails-views
│  ├─ Stimulus controllers, Turbo frames? → rails-stimulus-turbo
│  ├─ Forms, helpers? → rails-views
│  └─ Real-time updates, interactivity? → rails-stimulus-turbo
│
├─ 💼 Business Logic
│  ├─ Need to extract from controller/model? → rails-service
│  ├─ Complex workflow, multi-step process? → rails-service
│  ├─ Where should this logic go? → Rails Architecture (Skill)
│  └─ Planning how to structure? → rails-architect
│
├─ ⚙️ Background Jobs
│  ├─ Creating jobs, Sidekiq config? → rails-jobs
│  ├─ Job failing, debugging? → rails-jobs
│  └─ Performance optimization? → rails-jobs + Rails Performance Analyzer
│
├─ 🧪 Testing
│  ├─ Writing tests, test coverage? → rails-test
│  ├─ Fixing failing tests? → rails-test
│  └─ Test strategy planning? → rails-architect
│
├─ 🔒 Security
│  ├─ Security audit, vulnerabilities? → Rails Security Auditor (Skill)
│  ├─ Authentication setup? → rails-controller (Devise)
│  └─ Authorization implementation? → rails-controller (Pundit)
│
├─ 📈 Performance
│  ├─ Slow queries, N+1 problems? → Rails Performance Analyzer (Skill)
│  ├─ Caching strategy? → rails-architect + Rails Performance Analyzer
│  └─ Production performance issues? → rails-devops + Rails Performance Analyzer
│
├─ 🔧 Code Quality
│  ├─ Code smells, refactoring? → Ruby Refactoring Expert (Skill)
│  ├─ Architectural decisions? → rails-architect + Rails Architecture (Skill)
│  └─ Design patterns, best practices? → Rails Architecture (Skill)
│
├─ 🚀 DevOps & Deployment
│  ├─ CI/CD pipelines? → rails-devops
│  ├─ Docker, Kubernetes? → rails-devops
│  ├─ Production issues? → rails-devops
│  └─ Monitoring, logging? → rails-devops
│
├─ 🔄 Upgrades & Migrations
│  ├─ Rails version upgrade? → Rails Upgrade Assistant (Skill)
│  ├─ Deprecation warnings? → Rails Upgrade Assistant (Skill)
│  └─ Breaking changes? → Rails Upgrade Assistant (Skill)
│
├─ 🎯 GraphQL
│  ├─ GraphQL schema, types? → rails-graphql
│  ├─ Resolvers, mutations? → rails-graphql
│  └─ GraphQL N+1 queries? → rails-graphql
│
└─ 🏗️ Architecture & Planning
   ├─ Planning new feature? → rails-architect
   ├─ Design decisions? → rails-architect + Rails Architecture (Skill)
   ├─ Refactoring strategy? → rails-architect + Ruby Refactoring Expert
   └─ "Where should this go?" → Rails Architecture (Skill)
```

## Detailed Agent Guide

### Backend Layer

#### rails-model (Blue)
**Use when you need to**:
- Create or modify ActiveRecord models
- Design database schema and migrations
- Define associations (has_many, belongs_to, etc.)
- Add validations
- Create scopes or query methods
- Work with callbacks

**Example prompts**:
- "Create a Product model with inventory tracking"
- "Add a polymorphic association for comments"
- "Write a migration to add indexes"

---

#### rails-controller (Indigo)
**Use when you need to**:
- Create or modify controllers
- Implement RESTful actions (index, show, create, etc.)
- Handle authentication and authorization
- Manage strong parameters
- Define routes
- Work with Pundit policies

**Example prompts**:
- "Create a controller for managing orders"
- "Add Pundit authorization to the posts controller"
- "Implement bulk actions in the controller"

---

#### rails-service (Purple)
**Use when you need to**:
- Extract business logic from controllers or models
- Create service objects for complex operations
- Implement command or query patterns
- Handle multi-step workflows
- Coordinate operations across multiple models

**Example prompts**:
- "Extract payment processing logic into a service"
- "Create a service object for user onboarding workflow"
- "Implement order fulfillment service"

---

### Frontend Layer

#### rails-views (Green)
**Use when you need to**:
- Create or modify ERB templates
- Build ViewComponents
- Style with TailwindCSS
- Create forms with form helpers
- Work with layouts and partials
- Handle view-specific localization

**Example prompts**:
- "Create a view for displaying product details"
- "Convert this partial into a ViewComponent"
- "Build a form for creating orders"

---

#### rails-stimulus-turbo (Teal)
**Use when you need to**:
- Create Stimulus controllers
- Implement Turbo Frames or Streams
- Add JavaScript interactivity
- Build real-time features without page reloads
- Implement progressive enhancement
- Work with Hotwire

**Example prompts**:
- "Add a dropdown menu with Stimulus"
- "Implement real-time comments with Turbo Streams"
- "Create an autocomplete search with Stimulus"

---

### Background Processing

#### rails-jobs (Orange)
**Use when you need to**:
- Create ActiveJob or Sidekiq jobs
- Configure job queues and priorities
- Schedule recurring jobs
- Debug failing jobs
- Optimize job performance
- Handle job error scenarios

**Example prompts**:
- "Create a job to send daily email reports"
- "Debug why emails jobs are failing"
- "Set up recurring data cleanup job"

---

### API Layer

#### rails-graphql (Cyan)
**Use when you need to**:
- Design GraphQL schemas
- Create types, queries, mutations
- Implement resolvers
- Handle GraphQL authorization
- Optimize GraphQL queries (N+1)
- Set up subscriptions

**Example prompts**:
- "Create a GraphQL mutation for creating posts"
- "Fix N+1 queries in the users resolver"
- "Add GraphQL subscriptions for real-time updates"

---

### Quality & Testing

#### rails-test (Yellow)
**Use when you need to**:
- Write unit, integration, or system tests
- Review test coverage
- Fix failing tests
- Set up test fixtures or factories
- Implement TDD workflows
- Work with RSpec or Minitest

**Example prompts**:
- "Create tests for the Order model"
- "Review test coverage for the payments controller"
- "Write system tests for the checkout flow"

---

### Architecture & Planning

#### rails-architect (Gray)
**Use when you need to**:
- Plan architecture for new features
- Make design decisions
- Evaluate refactoring approaches
- Decide where logic should live
- Establish patterns for the team
- Think through database design

**Example prompts**:
- "I need to add multi-tenancy. What's the best approach?"
- "Should this logic be in a model or service?"
- "Plan the architecture for a subscription billing system"

**Note**: This is a **planning agent** - use before implementation!

---

### DevOps & Infrastructure

#### rails-devops (Red)
**Use when you need to**:
- Set up CI/CD pipelines
- Configure Docker or Kubernetes
- Deploy to production
- Monitor and log applications
- Optimize production performance
- Troubleshoot infrastructure issues
- Set up Kamal, Heroku, AWS, etc.

**Example prompts**:
- "Set up GitHub Actions for automated testing"
- "Create a Dockerfile for this Rails app"
- "The production app is slow, help me optimize it"

---

## Autonomous Skills

These Skills are automatically invoked by Claude based on your questions. You don't need to explicitly call them.

### Ruby Refactoring Expert
**Automatically triggered by mentions of**:
- "code smell", "refactor", "code quality"
- "technical debt", "complexity", "maintainability"
- "clean code", "SOLID", "DRY"
- "improve code", "simplify"
- "extract method", "extract class"

**What it does**:
- Identifies code smells
- Suggests refactoring patterns
- Provides before/after examples
- Prioritizes improvements

---

### Rails Architecture
**Automatically triggered by mentions of**:
- "architecture", "design pattern"
- "service object", "model organization"
- "multi-tenant", "authorization"
- "where should this go", "best practice"
- "Rails way", "convention"
- "separation of concerns", "domain model"

**What it does**:
- Provides architectural guidance
- Explains design patterns
- Helps make decisions about code organization
- Offers best practice recommendations

---

### Rails Performance Analyzer
**Automatically triggered by mentions of**:
- "slow", "performance", "N+1"
- "query optimization", "bottleneck"
- "speed up", "optimize"
- "caching", "memory usage"
- "database performance", "page load time"

**What it does**:
- Identifies performance bottlenecks
- Detects N+1 queries
- Suggests caching strategies
- Recommends database optimizations

---

### Rails Security Auditor
**Automatically triggered by mentions of**:
- "security", "vulnerability"
- "SQL injection", "XSS", "CSRF"
- "secure", "exploit", "attack"
- "mass assignment", "sanitize"
- "sensitive data", "encryption"

**What it does**:
- Scans for security vulnerabilities
- Identifies SQL injection risks
- Checks authorization implementation
- Reviews authentication patterns

---

### Rails Upgrade Assistant
**Automatically triggered by mentions of**:
- "upgrade Rails", "Rails version"
- "deprecation", "migration guide"
- "breaking change", "upgrade to Rails"
- "Rails 6 to 7", "Rails 7 to 8"
- "compatibility", "version bump"

**What it does**:
- Creates Rails upgrade plans
- Identifies breaking changes
- Provides version-specific migration guides
- Suggests gem compatibility updates

---

## When to Use Multiple Agents

Some tasks naturally involve multiple agents working together:

### New Feature Implementation
1. **rails-architect**: Plan the feature architecture
2. **rails-model**: Create models and migrations
3. **rails-controller**: Build controllers
4. **rails-views**: Create views
5. **rails-test**: Add comprehensive tests (automatic)

### Performance Investigation
1. **Rails Performance Analyzer**: Identify bottlenecks (automatic)
2. **rails-model**: Add database indexes
3. **rails-controller**: Implement caching
4. **rails-devops**: Set up monitoring

### Security Review
1. **Rails Security Auditor**: Scan for vulnerabilities (automatic)
2. **rails-controller**: Fix authorization issues
3. **rails-model**: Add data validations
4. **rails-test**: Add security tests

### Code Refactoring
1. **Ruby Refactoring Expert**: Identify code smells (automatic)
2. **rails-architect**: Plan refactoring approach
3. **rails-service**: Extract business logic
4. **rails-test**: Ensure tests still pass

## Still Not Sure?

If you're still uncertain which agent to use:

1. **Just ask naturally** - Claude will route to the appropriate agent
2. **Mention the context** - "I'm working on the checkout controller..."
3. **Start with rails-architect** - For planning and decision-making
4. **Let Skills work automatically** - They'll trigger when relevant keywords are mentioned

## Quick Reference Table

| Task | Agent/Skill | Color |
|------|-------------|-------|
| Models, DB schema | rails-model | Blue |
| Controllers, routes | rails-controller | Indigo |
| Views, templates | rails-views | Green |
| Stimulus, Turbo | rails-stimulus-turbo | Teal |
| Service objects | rails-service | Purple |
| Background jobs | rails-jobs | Orange |
| Testing | rails-test | Yellow |
| GraphQL | rails-graphql | Cyan |
| Architecture planning | rails-architect | Gray |
| DevOps, deployment | rails-devops | Red |
| Refactoring | Ruby Refactoring Expert | Auto |
| Performance | Rails Performance Analyzer | Auto |
| Security | Rails Security Auditor | Auto |
| Upgrades | Rails Upgrade Assistant | Auto |
| Architecture advice | Rails Architecture | Auto |

---

**Pro Tip**: The plugin is designed to work proactively. You don't always need to specify which agent to use - Claude will select the appropriate agent based on your request and the context of your conversation.
