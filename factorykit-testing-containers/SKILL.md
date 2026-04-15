---
name: factorykit-testing-containers
description: Guidance for testing code that uses FactoryKit shared containers, especially when container overrides interact with Swift Testing parallelization, task-local scoping, and FactoryTesting traits.
---

# FactoryKit Testing Containers

Use this skill when working on tests that:

- use `FactoryKit` for dependency registration or resolution
- use a custom `SharedContainer`
- override shared container registrations in tests
- use `FactoryTesting`
- run under Swift Testing and may execute in parallel

## Goal

Keep Factory container registrations isolated so tests do not leak overrides, cached values, or mocks across suites.

## Core Model

In FactoryKit, application dependencies are often resolved through a shared container type conforming to `SharedContainer`.

Common patterns include:

- a default `shared` container used by application code
- test-only registrations that override factories on that container
- scoped test helpers from `FactoryTesting`
- task-local shared container declarations using `@TaskLocal`

When the container is task-local, the current container value depends on the active task scope. That is what allows test-scoped overrides to work safely in parallel, but only if the tests use the proper scoped override mechanism.

## Required Rules

1. If a test mutates registrations on `ContainerType.shared`, isolate the suite or test with `FactoryTesting` container scoping such as `ContainerTrait<ContainerType>`.
2. Prefer a trait or scoped override helper that binds the container through the projected task-local value, such as `ContainerType.$shared`, when the container is task-local.
3. Reset the current test container before registering factories if the container manager supports reset semantics.
4. Register test factories before constructing the system under test.
5. Create fresh mocks, spies, and mutable backing state per test.
6. Keep registration, SUT creation, and execution within the same actor/task scope when dependencies are resolved from the shared container.
7. Avoid `Task.detached` for code paths that must inherit the current Factory container override.
8. Do not cache `ContainerType.shared` in static mutable state shared across tests.
9. Do not modify app-wide default registrations just to support tests when a scoped test container is available.

## Preferred Pattern

For a task-local shared container, prefer a shared trait helper like:

```swift
extension Trait where Self == ContainerTrait<AppContainer> {
    public static var appContainer: ContainerTrait<AppContainer> {
        .init(shared: AppContainer.$shared, container: .init())
    }
}
```

Then use the helper at the suite:

```swift
@Suite("Feature/ViewModel", .appContainer)
struct FeatureViewModelTests { ... }
```

Inside each test:

```swift
let container = AppContainer.shared
container.reset()
container.someDependency.register { mock }

let sut = FeatureViewModel()
```

## Factory-Specific Review Checklist

When reviewing Factory-based tests, check for:

- shared container mutation without `ContainerTrait`
- missing `reset()` before registrations
- registrations installed after SUT creation
- reuse of one mock across parallel tests
- container overrides hidden in global setup
- detached tasks relying on `ContainerType.shared`
- tests assuming the default shared container is isolated when it is not

## Common Failure Modes

### Tests interfere with each other in parallel

Likely cause:
- multiple suites mutate the same shared Factory container without scoped isolation

Next move:
- introduce `ContainerTrait<ContainerType>` and move all registrations into that scope

### The SUT resolves the wrong dependency

Likely cause:
- registration happened after SUT construction, or the lookup occurred outside the inherited task scope

Next move:
- register first, construct second, and verify task-local inheritance

### Overrides appear to disappear during async work

Likely cause:
- dependency resolution happens in detached or escaping work that does not inherit the current container override

Next move:
- inject the dependency directly or keep the work inside inherited task scope

## Framework Notes

- `FactoryTesting` exists specifically to support scoped container isolation in tests. Use it instead of hand-rolling global registration cleanup where possible.
- If your `SharedContainer.shared` is backed by `@TaskLocal`, `ContainerType.shared` is the current value and `ContainerType.$shared` is the binding handle used for scoped overrides.
- Treat the default shared container as application runtime state, not as a safe shared fixture for parallel tests.

## Default Recommendation

If a Factory-based test overrides anything on a shared container:

- scope the container per suite/test
- reset before registration
- register before creating the SUT
- keep execution inside inherited task scope
- avoid detached tasks or inject dependencies explicitly
