---
name: swift-testing-task-local-di
description: Guidance for tests that use task-local or ambient dependency containers under Swift Testing, especially when suites run in parallel. Covers container scoping, inherited task-local context, detached task hazards, setup ordering, and review heuristics.
---

# Swift Testing Task-Local DI

Use this skill when working on tests that:

- read dependencies from a shared or ambient DI container
- register or override dependencies during a test
- use task-local container state or scoped dependency overrides
- run under Swift Testing and may execute in parallel
- fail intermittently due to cross-test dependency leakage

## Goal

Keep dependency registration and resolution isolated per test or suite so parallel execution does not leak state across tests.

## Core Model

Many DI systems expose a shared or ambient container that feels global at call sites. In modern Swift code, that shared value may actually be task-local or dynamically scoped.

That distinction matters:

- A true global shared container is process-wide mutable state.
- A task-local shared container has a default value, but a test can override it for the current task scope.
- Child tasks usually inherit task-local values.
- Detached tasks do not inherit task-local values.

Tests that mutate ambient container state must be written to respect those scoping rules.

## When To Apply This Skill

Apply this skill if any of the following are true:

- A test registers mocks/stubs into a shared container.
- The code under test resolves dependencies lazily from ambient state.
- Tests are being migrated from XCTest to Swift Testing.
- A suite passes individually but fails under parallel execution.
- Different tests appear to overwrite each other's dependency registrations.
- Async work inside the SUT may escape the test's task scope.

## Required Rules

1. If a test mutates ambient container registrations, isolate the container per suite or per test.
2. Prefer an explicit test-scoped container override mechanism over mutating the default shared container directly.
3. Create fresh test doubles and fresh mutable backing state per test unless shared state is the behavior under test.
4. Reset or rebuild the test container before registering dependencies.
5. Register dependencies before constructing the system under test if the SUT resolves dependencies during initialization or first use.
6. Keep container setup, SUT creation, and execution within the same actor/task scope when dependency lookup is ambient.
7. Avoid `Task.detached` in code paths that must inherit test dependency overrides.
8. If detached or escaping work is required, inject dependencies explicitly into that work instead of relying on ambient lookup.
9. Do not cache shared-container instances in globals, static mutable fixtures, or cross-test singletons.
10. Prefer direct dependency injection over ambient container lookup when async boundaries make inheritance unclear.

## Preferred Workflow

### 1. Identify the container model

Determine which of these you are dealing with:

- true process-global container
- task-local shared container
- thread-local or dynamically scoped container
- explicit dependency object passed through constructors

Do not assume "shared" means global.

### 2. Find the override boundary

Look for:

- test traits
- scoped override APIs
- `withValue`-style task-local bindings
- per-test container constructors
- framework-specific test helpers

If the framework provides an isolation primitive, use it instead of inventing a new one.

### 3. Build the container inside test scope

Inside the active test scope:

- create a fresh container or reset the scoped one
- register only the dependencies the test needs
- create the SUT after registration
- run the assertion path before the scoped override ends

### 4. Verify async inheritance assumptions

Check whether the SUT:

- creates child tasks
- launches detached tasks
- stores closures for later execution
- hops onto different isolation domains
- resolves dependencies lazily after the test's setup phase

If any of these are true, confirm whether the override is still visible at the point of resolution.

## Review Checklist

When reviewing tests or async code that uses ambient DI, check for:

- tests mutating shared container state without scoped isolation
- missing container reset/rebuild before registrations
- SUT created before dependencies are registered
- one mutable mock reused across parallel tests
- detached tasks relying on ambient container lookup
- helper methods that silently mutate a process-wide default container
- flaky timing that suggests work is escaping the test's scoped override
- assertions happening before async work that consumes the override has completed
- main-actor state mixed with nonisolated setup in ways that blur scope

## Anti-Patterns

- Registering mocks into one shared global container for all tests in a parallel suite
- Constructing the SUT first and overriding dependencies afterward
- Reusing a single mutable spy across multiple test cases
- Assuming child-task inheritance rules also apply to detached tasks
- Hiding container mutation inside broad global test setup without clear scope
- Using ambient DI for long-lived background work when explicit injection is available

## Heuristics For Choosing Ambient vs Explicit Injection

Prefer ambient container lookup when:

- the code already follows that pattern consistently
- the work stays within one well-defined task scope
- the framework provides a clean scoped override for tests

Prefer explicit injection when:

- work may outlive the test scope
- detached/background tasks are involved
- a dependency is critical to correctness and should be obvious at construction
- task-local inheritance is difficult to reason about
- tests are flaky because of hidden lookup timing

## Common Failure Modes

### Tests pass alone, fail in parallel

Likely cause:
- shared mutable container state or shared mocks leaking across tests

Next move:
- isolate the container per suite/test and create fresh mocks per test

### Dependency override appears ignored

Likely cause:
- SUT was created before registration, or lookup happened in a detached task

Next move:
- register before SUT creation and verify task inheritance behavior

### Flaky async assertions

Likely cause:
- async work consumes dependencies after the scoped override has ended

Next move:
- add explicit synchronization, await completion, or inject dependencies directly

### Random cross-test contamination

Likely cause:
- a mutable mock, cache, or singleton fixture is reused across tests

Next move:
- rebuild all mutable test state per test and avoid static fixtures

## Framework Notes

This skill is framework-agnostic, but common implementations include:

- custom DI containers with test-scoped overrides
- `@TaskLocal` shared containers
- trait-based Swift Testing container isolation
- resolver/factory frameworks with shared registration state

Adapt the workflow to the framework in use, but preserve the isolation rules.

## What To Produce

When applying this skill, provide:

1. The isolation model in play
2. The likely leakage or inheritance risk
3. The minimal safe setup pattern
4. Any async boundary that invalidates ambient lookup assumptions
5. Concrete rules the team can encode in local docs or helper APIs

## Default Recommendation

If tests mutate a shared ambient container and run in parallel, the default safe approach is:

- isolate the container per suite/test
- reset or rebuild it per test
- register dependencies before SUT creation
- keep execution inside the inherited task scope
- avoid detached tasks or explicitly inject dependencies into them
