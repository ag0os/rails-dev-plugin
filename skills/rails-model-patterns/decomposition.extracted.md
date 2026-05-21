# Model Decomposition — `extracted` axis

Models hold data, associations, validations, and trivial derived values. Non-trivial domain logic lives in service objects — see `rails-service-patterns`.

## Callbacks

Minimize callbacks. Reserve them for **data integrity** only — normalizing a field, maintaining a counter cache. Side effects (email, downstream updates, external calls) belong in service objects, never in lifecycle hooks.

## Concerns

Use concerns for behavior **genuinely shared across models**. Domain logic for a single model's workflow goes to a service object, not a concern — a concern that only one model includes is usually a service object in disguise.

## Decomposition fixes

- **Fat model (500+ lines)** → extract service objects for workflows; value objects for derived data; query objects for complex scopes.
- **Callback doing complex orchestration** → move the workflow into a service object. Any callback that remains guards data integrity only.
