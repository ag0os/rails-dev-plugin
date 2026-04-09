# Coding Best Practices

Corrective guidelines for code generation. These address common tendencies that produce over-engineered, fragile, or needlessly complex code. Apply your training knowledge of clean code, SOLID, and refactoring -- these directives steer your judgment on where that knowledge is most often misapplied.

## Resist Over-Engineering

- A function is not a class. Don't wrap stateless logic in a class just to give it a `call` or `execute` method.
- Don't name things after design patterns (`UserFactory`, `OrderBuilder`, `PaymentStrategy`). Name after domain purpose.
- Don't add extension points, configuration options, or abstraction layers unless the current task demands them. Solve today's problem.
- Three similar lines of code is better than a premature abstraction. Extract only when duplication appears 3+ times with the same intent.
- Start with the simplest data structure (hash/dict, tuple, record). Graduate to a class only when you need behavior, validation, or encapsulation -- not before.

## Separate What Changes From What Doesn't

- Push side effects (I/O, mutation, external calls) to the boundaries. Keep core logic pure and testable.
- Model variants as data (sum types, discriminated unions, enums with behavior) -- not as string flags with scattered conditionals.
- When the same type/status check appears in multiple places, centralize it with polymorphic dispatch or pattern matching. Don't scatter it.
- If a single feature change touches 4+ files across layers, the related logic is too scattered. Colocate things that change together.

## Respect Duplication Nuance

- Same code is not always real duplication. If two blocks look identical but would change for different reasons, leave them separate.
- When duplication is real, extract it. But don't DRY across boundaries (modules, services) just to eliminate surface similarity.

## Keep Coupling Shallow

- Don't reach through `a.b.c.d`. Callers should only talk to immediate collaborators.
- Don't pass parameters through functions just to relay them. If a param is only forwarded, the dependency graph needs restructuring.
- Group fields that always travel together into their own type rather than passing them individually.

## Refactoring Safety

- One structural change per commit. Verify tests pass after each step.
- Never change behavior and structure in the same commit.
- If test coverage doesn't exist, write characterization tests before refactoring.
