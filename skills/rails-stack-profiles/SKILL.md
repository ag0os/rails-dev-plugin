---
name: rails-stack-profiles
description: Detects a Rails project's architecture axes — logic placement (native vs extracted) and delivery (html vs api) — so other skills load profile-appropriate guidance without inline conditionals. Use when planning architecture or when a recommendation depends on where business logic lives or whether the app renders HTML or serves JSON. NOT for test framework, job backend, cache store, or auth library choices — those are orthogonal facts detected by project-conventions.
allowed-tools: Read, Grep, Glob
---

# Rails Stack Axes

Resolve a project's architecture **once per session**, as two independent binary axes. Other skills consume the resolved axes to load one flat, branch-free guidance file each — they never carry inline `omakase / service-oriented` conditionals.

See [axis-a.md](axis-a.md) and [axis-b.md](axis-b.md) for characteristic patterns and code per axis value.

## The Two Axes

A project is **not** one of three profiles. It is one value on each of two axes:

| Axis | Values | Question it answers |
|------|--------|---------------------|
| **A — logic placement** | `native` \| `extracted` | Where does non-trivial business logic live? |
| **B — delivery** | `html` \| `api` | Does the surface render HTML or serve JSON? |

- **`native`** — logic lives in models, concerns, and POROs. The Rails-omakase way. Callbacks are acceptable for simple side effects.
- **`extracted`** — service/command objects are the default home for non-trivial logic. Common in larger teams and complex domains.
- **`html`** — server-rendered views, Hotwire (Turbo + Stimulus).
- **`api`** — headless JSON, serializers, no `app/views/` (mailer views aside).

The axes are independent. All four combinations are valid, including `native + api` (a Rails 8 omakase-style API-only app — a combination the old three-profile enum could not express).

The legacy names map cleanly: `omakase` = `native + html`, `service-oriented` = `extracted + html`, `api-first` = `extracted + api`.

## What is NOT an axis

Test framework, test data strategy, job backend, cache store, and auth library are **orthogonal project facts**, not architecture axes. A project with extracted services can still use Minitest; an omakase app can run Sidekiq. Inferring these from architecture is the central mistake this restructure removes.

Detect them directly via the **`project-conventions`** fingerprint. Skills consume those fingerprint fields; they never branch on a "profile" for them.

## Detecting Axis A — logic placement

| Signal | Points to |
|--------|-----------|
| `app/services/` with ~5+ files | `extracted` |
| Custom `ApplicationService` / `BaseService` base class | `extracted` |
| `gem "dry-monads"`, `gem "interactor"`, `gem "trailblazer"` | `extracted` |
| `app/services/` absent or near-empty | `native` |
| Domain logic in models + `app/models/concerns/` | `native` |
| Callbacks orchestrating multi-step workflows | `native` |
| POROs for operations, often in `app/models/` | `native` |

Weight the dominant pattern. A project with 50 service classes is `extracted` even if a few models are still fat.

## Detecting Axis B — delivery

| Signal | Points to |
|--------|-----------|
| App base controller is `ActionController::API` | `api` |
| No `app/views/` (or mailer views only) | `api` |
| `gem "jwt"`, `rack-cors`, serializer gems and no ERB | `api` |
| `app/views/**/*.erb` present | `html` |
| Hotwire (`turbo-rails`, `stimulus-rails`), `turbo_stream` responses | `html` |

## Detection procedure

```
1. Glob app/services/**/*.rb        → count → Axis A
2. Grep Gemfile for dry-monads / interactor / trailblazer → Axis A
3. Grep app/ for class Application\w+ < / class Base\w+ <  → Axis A
4. Read the app base controller class → ActionController::API? → Axis B
5. Glob app/views/**/*.erb           → presence → Axis B
6. Grep Gemfile for turbo-rails / jwt → Axis B
```

## Hybrids — resolve per surface, not per repo

Most real projects are mixed: an omakase app with an `extracted` billing module, or an `html` app with an `/api/v1` namespace. Do not force one repo-wide answer.

**Resolve the axes for the module or namespace in focus**, not the whole codebase. The directory being worked on (`app/billing/`, `app/controllers/api/`) has one clear answer on each axis. Ask the sharp question — "what is *this* surface?" — not the blurry one.

## Output contract

When reporting resolved axes:

```
## Stack Axes
- **logic:** [native | extracted]  — resolved from: [signal]
- **delivery:** [html | api]       — resolved from: [signal]
- **scope:** [whole app | namespace/module in focus]
```

Cache this for the session. Other skills read it; they do not re-detect.

## Degradation

If the axes cannot be resolved (empty or greenfield project), default to `native + html` (the Rails default posture) and state the assumption in one line.
