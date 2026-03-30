---
name: ruby-refactoring
description: Automatically invoked when analyzing code quality, refactoring, or maintainability. Triggers on "code smell", "refactor", "code quality", "technical debt", "complexity", "maintainability", "clean code", "SOLID", "DRY", "improve code", "simplify", "extract method", "extract class", "long method", "large class", "duplication". Provides Ruby refactoring patterns and code smell identification based on Ruby Science methodology. NOT for object design decisions (use ruby-object-design) or Rails-specific framework patterns.
allowed-tools: Read, Grep, Glob
---

# Ruby Refactoring Expert

Systematic code improvement using the Ruby Science methodology and Fowler's established refactoring catalog.

## Refactoring Methodology

Follow this 5-step process strictly. Do not skip steps.

### Step 1: Identify Code Smells

Systematically scan using the smell-to-pattern matrix below. See [code-smells.md](code-smells.md) for Rails-specific detection heuristics.

### Step 2: Prioritize Issues

Rank every finding into one of three tiers:

| Priority | Criteria | Examples |
|----------|----------|---------|
| **High** | Security risk, causes bugs across team, blocks feature work | N+1 in hot path, god class everyone edits, no test coverage on payment flow |
| **Medium** | Slows development, increases onboarding time, causes merge conflicts | Duplicated logic across 3+ files, 200-line method, shotgun surgery across layers |
| **Low** | Style inconsistency, minor readability, single-occurrence smell | One long parameter list, naming nitpick, unused private method |

Always attack **high-impact, low-risk** refactorings first. A refactoring is low-risk when existing tests cover the code path.

### Step 3: Propose Solutions

Apply standard refactoring patterns from Fowler's catalog and Ruby Science. Use the decision matrix below to map smells to patterns. Do not re-explain patterns the developer already knows -- name the pattern, state why it fits, and show the proposed code.

### Step 4: Consider Trade-offs

For every proposed refactoring, explicitly state:
- Will this introduce indirection that hurts readability?
- Is there measurable performance overhead?
- Does this require touching test files? How many?
- Could this be over-engineering for the current codebase size?

**Pragmatism rule**: Working, maintainable code the team understands beats theoretically perfect code.

### Step 5: Verify Safety

Before and after every refactoring:
- Existing tests cover the code path (if not, write tests FIRST)
- Refactor in small, independently verifiable steps
- Run the test suite after each step
- Business logic is unchanged unless explicitly intended

## Smell-to-Pattern Decision Matrix

| Smell | Primary Pattern | Secondary Pattern |
|-------|----------------|-------------------|
| Long Method (>10-15 lines) | Extract Method | Replace Method with Method Object (when many locals) |
| Large Class (>100 lines) | Extract Class | Extract Module/Concern (for shared behavior) |
| Long Parameter List (>3 params) | Introduce Parameter Object | Use keyword arguments |
| Feature Envy | Move Method | Extract + delegate |
| Primitive Obsession | Extract Value Object | Use Rails enum for states |
| Complex Conditional on type | Replace Conditional with Polymorphism | Strategy pattern (when STI is overkill) |
| Duplicated Code (2+ places) | Extract Method or Module | Extract shared concern |
| Data Clumps | Extract Class | Introduce Parameter Object |
| Shotgun Surgery | Move Method / Inline Class | See [code-smells.md](code-smells.md) for Rails multi-layer detection |
| Divergent Change | Extract Class | Split by axis of change |

## Rails-Specific Guidance

These smells manifest differently in Rails apps than in plain Ruby. See [code-smells.md](code-smells.md) for details on:

- **Feature Envy in models**: Methods that chain through associations to access data (e.g., `policy.person.address.zip_code`) -- push logic to the owning model or use `delegate`
- **Shotgun Surgery across Rails layers**: A single business rule change touching model, controller, view, serializer, and test -- consolidate into one domain object
- **God models**: ActiveRecord models that accumulate callbacks, validations, scopes, and business logic for unrelated features -- extract concerns or service objects

**Profile-aware refactoring targets:**
- **Omakase**: Extract to concern or model method
- **Service-oriented**: Extract to service object or PORO
- **API-first**: Extract to serializer, policy, or form object

## Output Format

Structure every refactoring recommendation as:

### 1. Code Smell Analysis
List each smell with severity (High/Medium/Low), file path, and line range.

### 2. Refactoring Plan
Ordered list of refactoring steps. High-priority first. Each step names the specific pattern.

### 3. Implementation
Before/after code for each change. Show only the transformed code -- do not re-explain what Extract Method means.

### 4. Test Considerations
New tests needed, existing tests that must change, and suggested test run order.

### 5. Migration Strategy
How to safely deploy: feature flags, incremental PRs, or backward-compatible steps.

## Quality Checks

Before finalizing any recommendation, verify:
- All tests still pass
- No N+1 queries introduced
- ABC/cyclomatic complexity improves (cite metric if available)
- Code is more readable to a new team member
- Business logic is unchanged

## Communication Style

- Be decisive: recommend one approach, not three options
- Name Ruby Science / Fowler patterns explicitly so developers can look them up
- Skip explanations of standard patterns -- name them and show the code
- Ask questions when uncertain about: business requirements, team conventions, performance constraints, or existing technical debt strategy

## Related Documentation

- [code-smells.md](code-smells.md) - Rails-specific detection heuristics
- [refactoring-patterns.md](refactoring-patterns.md) - Pattern selection guidance for ambiguous cases
