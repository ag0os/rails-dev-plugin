# Coding Best Practices: Avoiding Code Smells

Use these principles when writing or modifying code. They are language-agnostic and apply to any object-oriented or multi-paradigm codebase.

## Function / Method Design

- **Keep functions short.** If a function exceeds 10-15 lines, extract a well-named helper. Each function should do one thing.
- **Limit parameters to 3 or fewer.** When a function needs more, group related parameters into an object or named struct.
- **Don't pass data through functions just to relay it.** If a parameter is only forwarded to another call, the dependency graph needs restructuring.

## Class / Module Design

- **Keep classes focused.** A class exceeding ~100 lines likely has multiple responsibilities. Split along axes of change.
- **Don't create a class when a function will do.** If there's no state to manage, use a plain function or module.
- **Don't create single-method classes.** A class with only one public method is usually a function in disguise -- unless it genuinely needs constructor-injected dependencies.
- **Name after purpose, not mechanism.** Avoid names like `UserFactory`, `OrderBuilder`, `PaymentStrategy`. Name after what it represents in the domain, not the design pattern it implements.
- **Objects must be valid after construction.** Never require multi-step setup ceremonies. If an object can exist in an invalid state, fix the constructor.

## Data Modeling

- **Don't use primitives for domain concepts.** Wrap meaningful values (money, email, coordinates, status) in dedicated types with behavior and validation.
- **Prefer immutable value objects for data.** Use your language's immutable record, struct, or data class for values that don't need to change.
- **Graduate data structures incrementally.** Start simple (hash/dict/map), move to named struct/record, then full class. Only upgrade when you need behavior, validation, or encapsulation.
- **Group data that travels together.** If the same 3+ fields appear in multiple function signatures or classes, they belong in their own object.

## Conditional Logic

- **Replace complex conditionals with polymorphism.** If you switch on type or status in multiple places, use polymorphic dispatch instead.
- **Flatten nested conditionals.** Use guard clauses and early returns. Each nesting level adds cognitive load.
- **Centralize state-dependent behavior.** Don't scatter status or type checks across the codebase. Handle them in one place.

## Code Duplication and Coupling

- **Extract duplicated logic, but only when the duplication is real.** Same code with different reasons to change is not true duplication -- leave it alone.
- **Move logic to where the data lives.** If a function mostly accesses another object's data, it belongs on that object.
- **Minimize cross-cutting changes.** If adding a single feature requires touching 4+ files across different layers, the related logic is too scattered. Consolidate it.
- **Don't chain through objects.** Accessing `a.b.c.d` couples you to the entire chain. Delegate or encapsulate so callers only talk to their immediate collaborators.

## Refactoring Discipline

- **One refactoring per commit.** Never batch multiple structural changes together.
- **Ensure test coverage before refactoring.** If tests don't exist, write characterization tests first that capture current behavior.
- **Make the smallest possible change, then verify.** Run tests after each step. Small steps prevent compounding errors.
- **Never change behavior while refactoring.** Refactoring changes structure, not behavior. If you need both, do them in separate commits.

## Design Judgment

- **Don't over-abstract.** Three similar lines of code is better than a premature abstraction. Extract only when a pattern appears 3+ times with the same intent.
- **Don't design for hypothetical futures.** Solve today's problem. Speculative abstractions add complexity without value.
- **Prefer composition over inheritance.** Inheritance creates rigid hierarchies. Mix in behavior or inject collaborators instead.
- **Evaluate trade-offs explicitly.** Every refactoring adds indirection. Ask: does this improve readability, testability, or changeability enough to justify the added complexity?
