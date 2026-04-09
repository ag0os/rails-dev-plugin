# Coding Best Practices: Avoiding Code Smells

Use these principles when writing or modifying code. They are language-agnostic and paradigm-agnostic -- applicable to object-oriented, functional, and multi-paradigm codebases alike.

## Function / Method Design

- **Keep functions short.** If a function exceeds 10-15 lines, extract a well-named helper. Each function should do one thing.
- **Prefer pure functions.** Given the same inputs, return the same output with no side effects. Pure functions are easier to test, compose, and reason about.
- **Limit parameters to 3 or fewer.** When a function needs more, group related parameters into an object or named struct.
- **Don't pass data through functions just to relay it.** If a parameter is only forwarded to another call, the dependency graph needs restructuring.

## Structural Organization

- **Keep units of organization focused.** Whether a class, module, or namespace -- if it exceeds ~100 lines, it likely has multiple responsibilities. Split along axes of change.
- **Don't create a class when a function will do.** If there's no state to manage, use a plain function or module.
- **Don't create single-method classes.** A class with only one public method is usually a function in disguise -- unless it genuinely needs constructor-injected dependencies.
- **Name after purpose, not mechanism.** Avoid names like `UserFactory`, `OrderBuilder`, `PaymentStrategy`. Name after what it represents in the domain, not the design pattern it implements.
- **Objects must be valid after construction.** Never require multi-step setup ceremonies. If an object can exist in an invalid state, fix the constructor.
- **Push side effects to the edges.** Keep core logic pure. Perform I/O, mutation, and external calls at the boundaries of your system, not deep in business logic.

## Data Modeling

- **Don't use primitives for domain concepts.** Wrap meaningful values (money, email, coordinates, status) in dedicated types -- whether classes with methods or types with associated functions.
- **Prefer immutable data.** Use your language's immutable record, struct, data class, or frozen type for values that don't need to change. Return new values instead of mutating existing ones.
- **Model variants as data, not flags.** Use sum types, discriminated unions, or sealed hierarchies to represent mutually exclusive states. A `status` string with conditionals scattered everywhere is a code smell regardless of paradigm.
- **Graduate data structures incrementally.** Start simple (hash/dict/map), move to named struct/record/tuple, then full class or algebraic data type. Only upgrade when you need behavior, validation, or encapsulation.
- **Group data that travels together.** If the same 3+ fields appear in multiple function signatures or modules, they belong in their own type.

## Conditional Logic

- **Replace complex conditionals with dispatch.** If you switch on type or status in multiple places, use polymorphic dispatch (OOP) or pattern matching on sum types (FP) instead.
- **Flatten nested conditionals.** Use guard clauses and early returns. Each nesting level adds cognitive load.
- **Centralize state-dependent behavior.** Don't scatter status or type checks across the codebase. Handle them in one place.
- **Prefer data transformations over in-place mutation.** Pipeline data through a chain of transforms rather than modifying state in successive steps. This makes data flow explicit and each step independently testable.

## Code Duplication and Coupling

- **Extract duplicated logic, but only when the duplication is real.** Same code with different reasons to change is not true duplication -- leave it alone.
- **Colocate related logic.** In OOP, move methods to the class whose data they use. In FP, group related functions in the same module. Either way, keep things that change together close together.
- **Minimize cross-cutting changes.** If adding a single feature requires touching 4+ files across different layers, the related logic is too scattered. Consolidate it.
- **Minimize deep data access.** Reaching through `a.b.c.d` couples you to the entire structure. In OOP, delegate. In FP, destructure at the boundary or use accessor utilities. Callers should only know about their immediate dependencies.

## Refactoring Discipline

- **One refactoring per commit.** Never batch multiple structural changes together.
- **Ensure test coverage before refactoring.** If tests don't exist, write characterization tests first that capture current behavior.
- **Make the smallest possible change, then verify.** Run tests after each step. Small steps prevent compounding errors.
- **Never change behavior while refactoring.** Refactoring changes structure, not behavior. If you need both, do them in separate commits.

## Design Judgment

- **Don't over-abstract.** Three similar lines of code is better than a premature abstraction. Extract only when a pattern appears 3+ times with the same intent.
- **Don't design for hypothetical futures.** Solve today's problem. Speculative abstractions add complexity without value.
- **Prefer composition over inheritance.** Inheritance creates rigid hierarchies. Mix in behavior, compose functions, or inject collaborators instead.
- **Evaluate trade-offs explicitly.** Every refactoring adds indirection. Ask: does this improve readability, testability, or changeability enough to justify the added complexity?
