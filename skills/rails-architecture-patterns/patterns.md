# Architecture Patterns — Decision Frameworks

## STI vs Polymorphic Associations

**Decision checklist with specific thresholds:**

| Factor | STI | Polymorphic |
|--------|-----|-------------|
| Shared columns > 70% | Yes | -- |
| Shared columns < 30% | -- | Yes |
| Need to query all types together | Yes | Possible but slower |
| Types have different associations | -- | Yes |
| Type-specific columns | Few (< 5 nullable) | Many |

Follow standard Rails conventions for implementing whichever you choose.

## Logic placement and structural-smell fixes

Where business logic lives — and how to fix a god model, a fat controller, or callback hell — forks on **Axis A** (`rails-stack-profiles`). Read the matching decision guide:

- `native` → [decisions.native.md](decisions.native.md)
- `extracted` → [decisions.extracted.md](decisions.extracted.md)

## Monolith vs Engine vs Microservice

| Factor | Monolith | Engine | Microservice |
|--------|----------|--------|-------------|
| Team size | 1-10 | 5-20 | 10+ |
| Deploy independently | No | No | Yes |
| Code isolation | Directories | Gem boundary | Network boundary |
| Complexity | Low | Medium | High |
| When to choose | Default (always start here) | Large monolith needs internal boundaries | Proven need for independent scaling |

**Rule:** Start with a monolith. Only extract an engine when you have clear domain boundaries and shared code between apps. Only extract a microservice when you have a proven scaling bottleneck that can't be solved within the monolith.

## Premature Optimization Rule

```
1. First make it work (plain Rails)
2. Then make it right (refactor with data)
3. Then make it fast (optimize with measurements)
```

Don't add caching, service objects, engines, or microservices before measuring. YAGNI applies to architecture too.
