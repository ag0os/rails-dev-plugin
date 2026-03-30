---
name: project-conventions
description: Detects project-specific coding conventions by scanning the codebase — service object style, auth implementation, testing patterns, custom base classes, error handling, job conventions, serialization, frontend setup. Use when generating code that must match the project's existing patterns. Always check CLAUDE.md for intent that overrides detected conventions — CLAUDE.md documents where the project is going; the fingerprint documents where it is now. NOT for stack profile classification (use rails-stack-profiles).
allowed-tools: Read, Grep, Glob
---

# Project Conventions Detection

## Purpose

Auto-detect project-specific patterns so agents generate code matching the project, not generic Rails.
Produces a **Convention Fingerprint** — a structured summary of observed patterns.
Scan the codebase; do not ask the developer what patterns they use.

## Convention Categories

| Category | Detects | Key Agents |
|---|---|---|
| Service Objects | base class, entry point, naming, result type | rails-service, rails-architect |
| Auth | strategy, modules, custom controllers, token approach | rails-controller, rails-auth |
| Testing | framework specifics, shared examples, custom matchers, spec style | rails-test |
| Base Classes | custom ApplicationX classes | all agents |
| Error Handling | exception hierarchy, rescue_from patterns | rails-controller, rails-service |
| Jobs | base config, queue names, naming pattern, retry strategy | rails-jobs |
| Controllers | pagination, authorization, response helpers | rails-controller, rails-api |
| Domain Model | key entities, complexity, namespace structure | rails-model, rails-architect |
| Serialization | library, envelope format, naming | rails-api |
| Frontend | CSS framework, Stimulus patterns, component library, bundler | rails-views, rails-hotwire |

## CLAUDE.md Priority Note

Always check CLAUDE.md for intent that overrides detected conventions.
CLAUDE.md documents **where the project is going**; the fingerprint documents **where it is now**.

Example: if CLAUDE.md says "migrating from Devise to built-in auth", respect that over the detected Devise convention.

## Micro-Scan (for domain agents)

Each domain agent runs a lightweight 2-5 command scan for its relevant categories only.
Full detection commands in [detection-commands.md](detection-commands.md).

**Service Object detection example:**

```
1. Glob `app/services/**/*.rb` — if empty, skip
2. Read first 20 lines of 2-3 files to detect: base class, entry method (.call/.perform/.run), naming pattern
3. Grep for Result/Success/Failure to identify result type
4. Check Gemfile for `dry-monads`
```

## Convention Fingerprint Output Format

```
## Convention Fingerprint

**Services:** base=ApplicationService | entry=.call | result=ServiceResult | naming=Module::VerbNoun
**Auth:** strategy=devise | modules=[database_authenticatable,recoverable,trackable] | custom_controllers=yes
**Testing:** framework=rspec | data=factory_bot | style=request_specs | shared_examples=yes
**Base Classes:** ApplicationQuery, ApplicationForm, ApplicationDecorator
**Error Handling:** hierarchy=ApplicationError>ServiceError,ValidationError | rescue_from=yes
**Jobs:** base=ApplicationJob | queues=[default,mailers,critical] | naming=VerbNounJob | retry=3
**Controllers:** pagination=pagy | auth=pundit | response=respond_to
**Domain:** [top 5-8 entities] | namespaced=yes/no | models_count=N
**Serialization:** lib=alba | envelope={data:,meta:} | naming=ModelSerializer
**Frontend:** css=tailwind | stimulus=yes | components=ViewComponent | bundler=importmap
```

## Defaults (when undetectable)

For new or empty projects, assume these defaults:

| Category | Default |
|---|---|
| Service Objects | VerbNoun naming, .call, Struct-based Result |
| Auth | has_secure_password (Rails 8+) |
| Testing | Match test/ vs spec/ directory |
| Base Classes | None — use standard Rails bases |
| Error Handling | rescue_from in ApplicationController |
| Jobs | ApplicationJob, default queue |
| Controllers | Standard Rails patterns |
| Serialization | to_json or Jbuilder |
| Frontend | Importmap + Stimulus |

**Reference:** See [detection-commands.md](detection-commands.md) for complete detection recipes.
